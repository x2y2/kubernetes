## 创建搞可用的etcd集群

etc集群分布在三台机器上，这三台机器同时也作为node节点，先下载etcd文件，[点击这里](https://github.com/etcd-io/etcd/releases/download/v3.2.18/etcd-v3.2.18-linux-amd64.tar.gz)下载

从master复制ca.pem,kubernetes.pem,kubernetes-key.pem 到3个etcd节点的/data/kuernetes/ssl目录
```
cd /data/kuernetes/ssl
scp ca.pem kubernetes.pem kubernetes-key.pem node1:/data/kubernetes/ssl
scp ca.pem kubernetes.pem kubernetes-key.pem node2:/data/kubernetes/ssl
scp ca.pem kubernetes.pem kubernetes-key.pem node3:/data/kubernetes/ssl
```

解压下载的etcd文件，将etcd,etcdctl 文件复制到etcd节点的/data/kubernetes/bin目录

创建etcd的配置文件

```
mkdir /etc/etcd
vim /etc/etcd/etcd.conf
ETCD_LISTEN_PEER_URLS="https://0.0.0.0:2380"
ETCD_LISTEN_CLIENT_URLS="https://0.0.0.0:2379,https://0.0.0.0:4001"
ETCD_NAME="etcd3"
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://192.168.100.48:2380"
ETCD_ADVERTISE_CLIENT_URLS="https://192.168.100.48:2379"
ETCD_INITIAL_CLUSTER="etcd1=https://192.168.100.42:2380,etcd2=https://192.168.100.47:2380,etcd3=https://192.168.100.48:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_CERT_FILE="/etc/kubernetes/ssl/kubernetes.pem"
ETCD_KEY_FILE="/etc/kubernetes/ssl/kubernetes-key.pem"
ETCD_CLIENT_CERT_AUTH="true"
ETCD_TRUSTED_CA_FILE="/etc/kubernetes/ssl/ca.pem"
ETCD_PEER_CERT_FILE="/etc/kubernetes/ssl/kubernetes.pem"
ETCD_PEER_KEY_FILE="/etc/kubernetes/ssl/kubernetes-key.pem"
ETCD_PEER_CLIENT_CERT_AUTH="true"
ETCD_PEER_TRUSTED_CA_FILE="/etc/kubernetes/ssl/ca.pem"
```

创建etcd的systemd文件

```
/lib/systemd/system/etcd.service 
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
WorkingDirectory=/var/lib/etcd/
EnvironmentFile=-/etc/etcd/etcd.conf
# set GOMAXPROCS to number of processors
ExecStart=/bin/bash -c "GOMAXPROCS=$(nproc) /data/kubernetes/bin/etcd"

Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```
启动服务
```
systemctl daemon-reload
systemctl start etcd.service
systemctl status etcd.service
```
## etcd 集群健康检查
三个节点都这样操作，三个节点的etcd都启动好了后，用以下命令验证etcd集群。

```
etcdctl --endpoints="https://192.168.100.42:2379,https://192.168.100.47:2379,https://192.168.100.48:2379
" --ca-file=/data/kubernetes/ssl/ca.pem --cert-file=/data/kubernetes/ssl/kubernetes.pem   --key-file=/data/kubernetes/ssl/kubernetes-key.p
em cluster-health
```
出现如下所示，说明etcd集群是OK的
```
member 6e42f9ee8f2cb08d is healthy: got healthy result from https://192.168.100.48:2379
member c176747c58732c4d is healthy: got healthy result from https://192.168.100.42:2379
member d3126f94175c8ff7 is healthy: got healthy result from https://192.168.100.47:2379
cluster is healthy
```
查看etcd集群的member

```
etcdctl --endpoints="https://192.168.100.42:2379,https://192.168.100.47:2379,https://192.168.100.48:2379
" --ca-file=/data/kubernetes/ssl/ca.pem --cert-file=/data/kubernetes/ssl/kubernetes.pem   --key-file=/data/kubernetes/ssl/kubernetes-key.p
em member list   
6e42f9ee8f2cb08d: name=etcd3 peerURLs=https://192.168.100.48:2380 clientURLs=https://192.168.100.48:2379 isLeader=false
c176747c58732c4d: name=etcd1 peerURLs=https://192.168.100.42:2380 clientURLs=https://192.168.100.42:2379 isLeader=true
d3126f94175c8ff7: name=etcd2 peerURLs=https://192.168.100.47:2380 clientURLs=https://192.168.100.47:2379 isLeader=false
```

