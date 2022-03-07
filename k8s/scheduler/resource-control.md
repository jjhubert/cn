## 资源控制
- 磁盘控制(控制Pod对磁盘的使用容量)
  `定时获取容器的日志与当前容器可写层的磁盘情况，超过限制就驱逐。`
    - limits.ephemeral-storage
    - requests.ephemeral-stoage
- init容器的资源
    - 多个init容器以顺序执行。
    - 多个init容器，以其中占用资源最大的配置来向api-server进行申请。
- NodeRestriction(节点访问控制)
    - 该准入控制器限制了 kubelet 可以修改的 Node 和 Pod 对象。
    - kubelet 只可修改自己的 Node API 对象，只能修改绑定到节点本身的 Pod 对象。