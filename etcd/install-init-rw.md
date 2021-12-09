## Download and Install

```shell 
ETCD_VER=v3.4.17
DOWNLOAD_URL=https://github.com/etcd-io/etcd/releases/download
rm -f /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz
rm -rf /tmp/etcd-download-test && mkdir -p /tmp/etcd-download-test
curl -L ${DOWNLOAD_URL}/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz -o /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz
tar xzvf /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz -C /tmp/etcd-download-test --strip-components=1
rm -f /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz

# set env
export PATH=$PATH:/tmp/etcd-download-test/
echo export PATH=$PATH:/tmp/etcd-download-test/ >> /etc/profile
```

## Init
> What is the difference between listen-<client,peer>-urls, advertise-client-urls or initial-advertise-peer-urls?
>
> listen-client-urls and listen-peer-urls specify the local addresses etcd server binds to for accepting incoming connections. To listen on a port for all interfaces, specify 0.0.0.0 as the listen IP address.
>
> advertise-client-urls and initial-advertise-peer-urls specify the addresses etcd clients or other etcd members should use to contact the etcd server. The advertise addresses must be reachable from the remote machines. Do not advertise addresses like localhost or 0.0.0.0 for a production setup since these addresses are unreachable from remote machines.
```shell 
etcd --listen-client-urls 'http://localhost:12379' \
 --advertise-client-urls 'http://localhost:12379' \
 --listen-peer-urls 'http://localhost:12380' \
 --initial-advertise-peer-urls 'http://localhost:12380' \
 --initial-cluster 'default=http://localhost:12380'
```

## Member list
```shell 
etcdctl member list --write-out=table --endpoints=localhost:12379
```

## Read and write tests
- write
```shell 
etcdctl --endpoints=localhost:12379 put /key1 val1
etcdctl --endpoints=localhost:12379 put /key2 val2
```

- read
```shell 
etcdctl --endpoints=localhost:12379 get --prefix /
# only list keys
etcdctl --endpoints=localhost:12379 get --prefix / --keys-only
```

- set key information output using JSON
```shell 
etcdctl --endpoints=localhost:12379 get /key -wjson
```
```json
{"header":{"cluster_id":17478742799590499669,"member_id":14532165781622267127,"revision":7,"raft_term":2},"kvs":[{"key":"L2tleQ==","create_revision":4,"mod_revision":7,"version":4,"value":"dmFsNA=="}],"count":1}
```
> key and value are encoded by base64

- watch
```shell 
etcdctl --endpoints=localhost:12379 watch --prefix /
```
```shell 
etcdctl --endpoints=localhost:12379 put /key val1
etcdctl --endpoints=localhost:12379 put /key val2
etcdctl --endpoints=localhost:12379 put /key val3
etcdctl --endpoints=localhost:12379 put /key val4
```

- specify revision to watch
```shell 
etcdctl --endpoints=localhost:12379 watch --prefix / --rev 2
```