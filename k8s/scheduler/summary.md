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

### 


## 资源控制
- 磁盘控制(控制Pod对磁盘的使用容量)
`定时获取容器的日志与当前容器可写层的磁盘情况，超过限制就驱逐。`
	- limits.ephemeral-storage
	- requests.ephemeral-stoage
- init容器的资源
	- 多个init容器以顺序执行。
	- 多个init容器，以其中占用资源最大的配置来向api-server进行申请。

