# Eviction
- node-pressure eviction
  - eviction signals
  - eviction threshold
    - hard eviction threshold
    - soft eviction threshold
  - eviction monitoring interval
  - node condition
  - node condition oscillation
  - reclaiming node disk resources
  - pod selection for kubelet eviction
- api-initiated eviction

## node-presure eviction

### eviction signals
**驱逐信号**是指当前时间下特定资源的一个**状态**。Kubelet通过驱逐信号与驱逐阈值比较再做出驱逐决定，**驱逐决定取决于节点的最小可用资源数**。

#### eviction signals example
`驱逐信号的值可以是字面值也可以是百分比值。Kubelet收集Pod的数值是通过cgroupfs，而不是通过free这类型工具。`
- memory.available := node.status.capacity[memory] - node.stats.memory.workingSet
- nodefs.available := node.stats.fs.available
- nodefs.inodesFree := node.stats.fs.inodesFree
- imagefs.available := node.stats.runtime.imagefs.available
- imagefs.inodesFree := node.stats.runtime.imagefs.inodesFree
- pid.available := node.stats.rlimit.maxpid - node.stats.rlimit.curproc

### eviction threshold
驱逐阈值或者驱逐条件是一个逻辑表达式。通过**驱逐信号**、**逻辑运算符**、**字面值或比例值**组成驱逐条件的表达式。

`[eviction-signal][operator][quantity]`

example:
```
memory.available < 10% 
memory.available < 1Gi
```

#### soft eviction threshold
通过设置**宽恕时间**来让Pod在一定时间内运行，超过这段时间后仍然触发驱逐阈值将会驱逐Pod。
- eviction-soft : 设置驱逐阈值的表达式。一组驱逐设置例如: `memory.available<1.5Gi`
- eviction-soft-grace-period: 定义在触发Pod驱逐之前能保留多长时间。也是一组驱逐设置例如: `memory.available=1m30s`
- eviction-max-pod-grace-period: 定义发生驱逐事件后，Pod能最大保留多长时间。

ps: Kubelet根据`eviction-soft-grace-period`与`eviction-max-pod-grace-period`之间的值，选择较少的来应用。

### hard eviction threshold
没有宽恕时间，每当达到硬驱逐阈值时，Kubelet会立即驱逐Pod以释放资源。
- eviction-hard : 设置驱逐与之的表达式。 例如: `memory.available<1Gi`

### eviction monitoring interval
kubelet驱逐检查间隔(默认): `housekeeping-interval: 10s` 

### node condition
主要是通过磁盘、内存、PID数目的维度来判断节点压力，因为这些都是不可压缩资源。

`可以通过--node-status-update-frequency参数指定kubelet收集节点信息的频率。默认是10s。`

| Node Condition  | Eviction Signal  | Description |
| :----: | :----: | :----: |
| MemoryPressure  | memory.available  | Available memory on the node has satisfied an eviction threshold |
| DiskPressure  | nodefs.available, nodefs.inodesFree, imagefs.available, or imagefs.inodesFree  | Available disk space and inodes on either the node's root filesystem or image filesystem has satisfied an eviction threshold |
| PIDPressure  | pid.available  | Available processes identifiers on the (Linux) node has fallen below an eviction threshold |

### node condition oscillation
控制Kubelet需要等待多长时间才转换节点状态。避免节点的状态频繁切换。

`eviction-pressure-transition-period: 5m`

### reclaim node disk resources
磁盘主要存储镜像、Pod和容器的数据。镜像数据可以单独存放在imagefs的文件系统。因此在做资源回收的时候需要考虑是否有imagefs。
- 有加载imagefs，会出现两种情况。
  - 如果imagefs的剩余空间接近到达驱逐阈值，kubelet会清理没有使用的镜像。
  - 若nodefs剩余空间到达阈值，kubelet会回收死掉的Pod和容器。
- 没有加载imagefs，只出现一种情况。
  - 若nodefs剩余空间到达阈值，kubelet会清理没用的镜像、死掉的Pod和容器。

### pod selection for kubelet eviction
1. Pod的使用资源是否超过申请
2. Pod的优先级
3. Pod当前使用资源相较于申请

Pod根据以下排名规则被Kubelet进行驱逐
1. **BestEffort**或**Burstable**的Pod超过申请时的资源为第一个参考指标。然后根据优先级和超出申请多少的资源作为第二个参考指标。根据这两个指标先清理一部分Pod腾出资源空间。
2. 若还需要进一步进行清理，则根据**Guaranteed**和**Burstable**优先级以及哪些使用的资源比申请要少，再进行最后驱逐。