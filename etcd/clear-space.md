### set etcd storage size
```sh
etcd --quota-backend-bytes=$((16*1024*1024))
```

### check endpoint status
```shell
ETCDCTL_API=3 etcdctl --write-out=table endpoint status
```

### simulated input 
```shell
while [ 1 ]; do dd if=/dev/urandom bs=1024 count=1024 | ETCDCTL_API=3 etcdctl put key || break; done
```

### check alarm events
```shell
ETCDCTL_API=3 etcdctl alarm list
```

### clear fragment
```shell
ETCDCTL_API=3 etcdctl defrag
```

### clear alarm event
```shell
ETCDCTL_API=3 etcdctl alarm disarm
```

### keep one hour of history
```shell
etcd --auto-compaction-retention=1
```

### compress amounts of history version
```shell
etcdctl compact 3
```