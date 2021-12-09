### create 30 seconds of the lease
```shell 
etcdctl --endpoints=localhost:12379 lease grant 30
```

### check lease list
```shell
etcdctl --endpoints=localhost:12379 lease list
```

### bind the lease to the key
```shell
etcdctl --endpoints=localhost:12379 --lease 1cf77c6926d4620e put /a b
```

### the lease is renewed automatically
```shell
etcdctl --endpoints=localhost:12379 lease keep-alive 1cf77c6926d4620e
```