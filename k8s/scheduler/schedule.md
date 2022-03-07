## 调度

### kube-scheduler
- 做什么: 把Pod调度到响应的Node
- 怎么做: 
	1. 绑定Node
		- 监听api-server创建Pod的事件
		- 查询还有绑定Node的Pod
	2. 调度Pod到Node
        - 调度因素
            - 公平调度
            - 资源高利用
            - QOS
              - Guaranteed(request == limit)
              - Burstable(request < limit)
              - BestEffort(not config)
            - affinit & anti-affinity
            - 数据本地化（data locality）
            - 内部负载干绕（inter-workload interference）
            - deadlines

### 调度阶段
1. predicate（过滤不符合条件的节点）
	- 检查Node资源（CPU、Memory、Disk、Port）
	- 检查是否指定NodeName或NodeSelector
	- 检查Pod亲和性
	- 检查taint污点容忍
2. priority（优先级排序，选择高优先级）
	- 跨故障域。减少将Pod分配到同一个Node、Rack、Zone。
	- 优先分配资源使用少的节点。
	- 优先分配到Node亲和性的节点。
	- 优先分配到容忍taint的节点。

### pod分配方式
- nodeName: 指定pod调度到指定名称的节点
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
	- name: nginx
	  image: nginx
  nodeName: kube-01 
```
- nodeSelector: 指定pod调度到具有相同label的节点

`给node打上相应label`
```shell
kubectl label nodes node-01 tier=bff
```
`指定nodeSelector`
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    env: test
spec:
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
  nodeSelector:
    tier: bff
```
- nodeAffinity: 调度条件的核心围绕节点
  - 适用要求
    - requiredDuringSchedulingIgnoredDuringExecution : 与nodeSelector类似，必须调度到满足所有条件的节点。
    - preferredDuringSchedulingIgnoredDuringExecution : 尽量调度到满足条件的节点。

**备注**
```
1. 一个nodeAffinity 类型与多个 nodeSelectorTerms 关联，其中一个 nodeSelectorTerms 满足就调度到节点上。
2. 一个nodeSelectorTerms 类型与多个 matchExpressions 关联，则必须满足所有 matchExpressions 才能调度到节点上。
3. 亲和性只有在调度时候生效。
```
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: with-node-affinity
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: kubernetes.io/e2e-az-name
            operator: In
            values:
            - e2e-az1
            - e2e-az2
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: another-node-label-key
            operator: In
            values:
            - another-node-label-value
  containers:
  - name: with-node-affinity
    image: k8s.gcr.io/pause:2.0
```
- podAffinity: Pod亲和性。调度条件的核心围绕Pod
  - 适用要求
      - requiredDuringSchedulingIgnoredDuringExecution : 硬性规定
      - preferredDuringSchedulingIgnoredDuringExecution : 软性规定
      - 适用要求下配置多个选项是逻辑与的逻辑
- podAntiAffinity: Pod反亲和性。调度条件的核心围绕Pod。
    - topologyKey : 指定拓扑域（故障域）来对Pod的反亲和分配。
    - namespaceSelector : 根据label匹配相对应的namespace。
    - namespaces : 指定应用在哪些namespace。
    - 适用要求
      - preferredDuringSchedulingIgnoredDuringExecution : 软性规定
      - preferredDuringSchedulingIgnoredDuringExecution : 软性规定
      - 适用要求下配置多个选项是逻辑与的逻辑
      
**example**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: with-pod-affinity
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: security
            operator: In
            values:
            - S1
        topologyKey: topology.kubernetes.io/zone
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
            - key: security
              operator: In
              values:
              - S2
          topologyKey: topology.kubernetes.io/zone
  containers:
  - name: with-pod-affinity
    image: k8s.gcr.io/pause:2.0
```
```
Pod 亲和性规则表示，仅当节点和至少一个已运行且有键为“security”且值为“S1”的标签 的 Pod 处于同一区域时，才可以将该 Pod 调度到节点上。
Pod 反亲和性规则表示，如果节点处于 Pod 所在的同一可用区且具有键“security”和值“S2”的标签， 则该 pod 不应将其调度到该节点上。
```
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-cache
spec:
  selector:
    matchLabels:
      app: store
  replicas: 3
  template:
    metadata:
      labels:
        app: store
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - store
            topologyKey: "kubernetes.io/hostname"
      containers:
      - name: redis-server
        image: redis:3.2-alpine
```
```
将 web-server 的所有副本与具有 app=store 选择器标签的 Pod 不会有两个 Pod 被调度到同一节点上。
```
- taints & toleration

`Pod的toleration必须与Node的所有taint匹配才能被调度到Node上，缺一不可。taint与toleration类似一个过滤器，过滤不符合要求的节点。`
  - taint的effect
    - NoSchedule : 不调度到该Node(硬性规定)
    - PreferNoSchedule : 尽量不调度到该Node(软性规定)如果未被过滤的污点中不存在 effect值为NoSchedule的污点，但是存在effect值为PreferNoSchedule的污点，则Kubernetes会尝试不将Pod分配到该节点。
    - NoExecute : 不调度到该Node且当前Node下的Pod没有容忍该taint的话会直接被驱逐或在多少秒之后被驱逐
  - 对Node的taint管理
```shell
# 添加taint
kubectl taint nodes node1 key1=value1:NoSchedule
# 删除taint
kubectl taint nodes node1 key1=value1:NoSchedule-
```
  - 对Pod添加toleration
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    env: test
spec:
  containers:
	- name: nginx
	  image: nginx
	  imagePullPolicy: IfNotPresent
  tolerations:
	- key: "example-key"
	  operator: "Exists"
	  effect: "NoSchedule"
	- key: "key1"
	  operator: "Equal"
	  value: "value1"
	  effect: "NoSchedule"
```
`加入Pod保留时间。3600秒后被驱逐。`
```yaml
tolerations:
- key: "key1"
  operator: "Equal"
  value: "value1"
  effect: "NoExecute"
  tolerationSeconds: 3600
```




