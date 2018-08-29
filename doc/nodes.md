## flannel安装配置

下载flannel,解压将flanneld,mk-docker-opts.sh 复制到node节点的/data/kubernetes/bin

```
wget https://github.com/coreos/flannel/releases/download/v0.10.0/flannel-v0.10.0-linux-amd64.tar.gz
tar zxf flannel-v0.10.0-linux-amd64.tar.gz -C /usr/local/bin/
```

配置flannel

```
cat /data/dashboard/conf/flannel
FLANNEL_ETCD="-etcd-endpoints=https://192.168.100.42:2379,https://192.168.100.47:2379,https://192.168.100.48:2379"
FLANNEL_ETCD_KEY="-etcd-prefix=/kubernetes/network/"
FLANNEL_ETCD_CAFILE="-etcd-cafile=/data/kubernetes/ssl/ca.pem"
FLANNEL_ETCD_CERTFILE="-etcd-certfile=/data/kubernetes/ssl/kubernetes.pem"
FLANNEL_ETCD_KEYFILE="-etcd-keyfile=/data/kubernetes/ssl/kubernetes-key.pem"
```
- 需要的证书见cert.md

设置flanneld系统服务


```
cat /lib/systemd/system/flanneld.service
[Unit]
Description=Flanneld
Documentation=https://github.com/coreos/flannel
After=network.target
Before=docker.service
[Service]
EnvironmentFile=-/data/kubernetes/conf/flannel
ExecStartPre=/data/kubernetes/bin/remove-docker0.sh
ExecStart=/data/kubernetes/bin/flanneld ${FLANNEL_ETCD} ${FLANNEL_ETCD_KEY} ${FLANNEL_ETCD_CAFILE} ${FLANNEL_ETCD_CERTFILE} ${FLANNEL_ETCD_KEYFILE}
ExecStartPost=/data/kubernetes/bin/mk-docker-opts.sh -d /etc/default/docker -c
Restart=on-failure
Type=notify
LimitNOFILE=65536
[Install]
WantedBy=multi-user.target
```

- remove-docker0.sh  从kubernetes-server-linux-amd64.tar.gz解压包中复制

```
cp kubernetes/cluster/centos/node/bin/remove-docker0.sh  /data/kubernetes/bin
```


Flannel CNI集成

```
wget https://github.com/containernetworking/plugins/releases/download/v0.7.1/cni-plugins-amd64-v0.7.1.tgz
tar zxf cni-plugins-amd64-v0.7.1.tgz -C /data/kubernetes/bin/cni
```

etc中创建网络配置（在etc节点上操作）

```
etcdctl  --ca-file=/data/kubernetes/ssl/ca.pem --cert-file=/data/kubernetes/ssl/kubernetes.pem --key-file=/data/kubernetes/ssl/kubernetes-key.pem --no-sync -C --endpoints="https://192.168.100.42:2379,https://192.168.100.47:2379,https://192.168.100.48:2379" mk /kubernetes/network/config '{"Network": "172.17.0.0/16","Backend": {"Type": "vxlan","VNI": 1}}'
```

启动flanneld
```
systemctl daemon-reload
systemctl enable falnneld.service
systemctl start flanneld.service
systemctl status flanneld.service
```

配置Docker使用flanneld
```
cat /lib/systemd/system/docker.service
[Unit]
Description=Docker Application Container Engine
Documentation=https://docs.docker.com
After=network.target docker.socket firewalld.service
Requires=docker.socket

[Service]
Type=notify
EnvironmentFile=-/etc/default/docker
ExecStart=/usr/bin/dockerd -H fd:// --insecure-registry=registry:5000 $DOCKER_OPTS
ExecReload=/bin/kill -s HUP $MAINPID
LimitNOFILE=1048576
LimitNPROC=infinity
LimitCORE=infinity
TasksMax=infinity
TimeoutStartSec=0
Delegate=yes
KillMode=process

[Install]
WantedBy=multi-user.target
```

- --insecure-registry=registry:5000  是用的私有镜像库，见registry.md

重启Docker

```
systemctl daemon-reload
systemctl restart docker
```

## CNI集成

安装CNI插件

```
wget https://github.com/containernetworking/plugins/releases/download/v0.7.1/cni-plugins-amd64-v0.7.1.tgz
mkdir /data/kubernetes/bin/cni
tar zxf cni-plugin-amd64-v0.7.1.tgz -C /data/kubernetes/bin/cni
```

配置CNI

```
cat /etc/cni/net.d/10-default.conf
{
        "name": "flannel",
        "type": "flannel",
        "delegate": {
            "bridge": "docker0",
            "isDefaultGateway": true,
            "mtu": 1400
        }
}
```

## kublet 配置

从解压的kubernetes-server包里将kubelet复制到各个node 节点的/data/kubernetes/bin目录

创建kubelet目录

mkdir /var/lib/kubelet

创建kubelet系统服务

```
cat /lib/systemd/system/kubelet.service  
[Unit]
Description=Kubernetes Kubelet Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=docker.service
Requires=docker.service
[Service]
WorkingDirectory=/var/lib/kubelet
ExecStart=/data/kubernetes/bin/kubelet \
  --pod-infra-container-image=registry:5000/pause-amd64:3.1 \
  --bootstrap-kubeconfig=/data/kubernetes/conf/bootstrap.kubeconfig \
  --kubeconfig=/data/kubernetes/conf/kubelet.kubeconfig \
  --cert-dir=/data/kubernetes/ssl \
  --network-plugin=cni \
  --cni-conf-dir=/etc/cni/net.d \
  --cni-bin-dir=/data/kubernetes/bin/cni \
  --cluster-dns=10.254.1.1 \
  --cluster-domain=cluster.local. \
  --allow-privileged=true \
  --fail-swap-on=false \
  --logtostderr=false \
  --v=2 \
  --log-dir=/data/kubernetes/logs
Restart=on-failure
KillMode=process
[Install]
WantedBy=multi-user.target
```

- 确保bootstrap.kubeconfig 文件已经存在
- kubelet会使用--bootstrap-kubeconfig指定的文件中的用户名和token向kube-apiserver发TLS Bootstrapping请求;
- --pod-infra-container-image 配置从私有镜像库下载pause-amd64

启动服务

systemctl start kubelet

在master节点观察csr状态

kubectl get csr

此时的csr为Pending状态

批准kubelet TLS证书请求
```
kubectl get csr|grep 'Pending' | awk 'NR>0{print $1}'| xargs kubectl certificate approve
```

- kube-apiserver 接受csr请求后，kubelet会在--cert-dir 目录下自动生成证书（kubelet-client.crt 和kubelet-client.key），并保存到—kubeconfig指定的文件中
- 将node加入集群

## 配置kube-proxy

复制kube-proxy 到node节点/data/kubernetes/bin

配置kube-proxy使用LVS

apt install -y ipvsadm ipset conntrack

kube-proxy 系统服务

```
cat /lib/systemd/system/kube-proxy.service
[Unit]
Description=Kubernetes Kube-Proxy Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target
[Service]
WorkingDirectory=/var/lib/kube-proxy
ExecStart=/data/kubernetes/bin/kube-proxy \
  --kubeconfig=/data/kubernetes/conf/kube-proxy.kubeconfig \
  --masquerade-all \
  --feature-gates=SupportIPVSProxyMode=true \
  --proxy-mode=ipvs \
  --ipvs-min-sync-period=5s \
  --ipvs-sync-period=5s \
  --ipvs-scheduler=rr \
  --logtostderr=false \
  --v=2 \
  --log-dir=/data/kubernetes/logs
Restart=on-failure
LimitNOFILE=65536
[Install]
WantedBy=multi-user.target
```

- 确保kube-proxy.kubeconfig文件存在
- kube-proxy.kubeconfig的创建见kubeconfig.md

启动服务

```
systemctl daemon-reload
systemctl enable kube-proxy
systemctl start kubelet kube-proxy
systemctl status kubelet kube-proxy
```
