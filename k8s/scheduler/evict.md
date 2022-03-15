# Eviction
- node-presure eviction
  - eviction signals
  - eviction threshold
    - hard eviction threshold
    - soft eviction threshold
  - eviction monitoring interval
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