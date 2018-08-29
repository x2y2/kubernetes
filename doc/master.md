
## master端软件安装

从[百度盘](https://pan.baidu.com/s/1FEDIPiP-tGBEXNUnepIykA)中下载kubernetes-sever-linux-amd64-1.10.0.tar.gz,解压后将包中的kubectl,kube-apiserver,kube-controller-manager,kube-scheduler文件复制到/data/kubernetes/bin,对kubectl 建软链到/usr/local/bin

```
cp kubectl kube-apiserver kube-controller-manager kube-scheduler /data/kubernetes/bin
ln -s /data/kubernetes/bin/kubectl /usr/local/bin/kubectl
```

## kube-apiserver 

master 所需要的证书见cert.md说明

kube-apiserver 服务文件

```
cat /lib/systemd/system/kube-apiserver.service 
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target
[Service]
ExecStart=/data/kubernetes/bin/kube-apiserver \
 	    --enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota,NodeRestriction \
 	    --bind-address=192.168.100.39 \
            --external-hostname=192.168.100.39 \
 	    --authorization-mode=Node,RBAC \
 	    --runtime-config=rbac.authorization.k8s.io/v1 \
 	    --anonymous-auth=false \
 	    --kubelet-https=true \
 	    --basic-auth-file=/data/kubernetes/conf/basic-auth.csv \
 	    --enable-bootstrap-token-auth \
 	    --token-auth-file=/data/kubernetes/conf/bootstrap-token.csv \
            --service-cluster-ip-range=10.254.0.1/16 \
            --service-node-port-range=20000-40000 \
            --tls-cert-file=/data/kubernetes/ssl/kubernetes.pem \
            --tls-private-key-file=/data/kubernetes/ssl/kubernetes-key.pem \
            --client-ca-file=/data/kubernetes/ssl/ca.pem \
            --service-account-key-file=/data/kubernetes/ssl/ca-key.pem \
            --etcd-cafile=/data/kubernetes/ssl/ca.pem \
            --etcd-certfile=/data/kubernetes/ssl/kubernetes.pem \
            --etcd-keyfile=/data/kubernetes/ssl/kubernetes-key.pem \
            --etcd-servers="https://192.168.100.42:2379,https://192.168.100.47:2379,https://192.168.100.48:2379" \
            --enable-swagger-ui=true \
            --allow-privileged=true \
            --audit-log-maxage=30 \
            --audit-log-maxbackup=3 \
            --audit-log-maxsize=100 \
            --audit-log-path=/data/kubernetes/logs/k8s/api-audit.log \
            --event-ttl=1h \
            --v=2 \
            --logtostderr=false \
            --log-dir=/data/kubernetes/logs
Restart=on-failure
Type=notify
LimitNOFILE=65536
[Install]
WantedBy=multi-user.target
```
```
cat /data/kubernetes/conf/basic-auth.csv
admin,admin,1
readonly,readonly,2
```

启动服务

```
systemctl daemon-reload
systemctl start kube-apiserver
ststemctl status kube-apiserver
```
## kube-controller-manager

kube-controller-manager服务文件
```
cat /lib/systemd/system/kube-controller-manager.service 
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
[Service]
ExecStart=/data/kubernetes/bin/kube-controller-manager \
 	    --address=127.0.0.1 \
 	    --master=http://127.0.0.1:8080 \
 	    --allocate-node-cidrs=true \
 	    --service-cluster-ip-range=10.254.0.1/16 \
            --cluster-cidr=172.17.0.0/16 \
            --cluster-name=kubernetes \
            --cluster-signing-cert-file=/data/kubernetes/ssl/ca.pem \
            --cluster-signing-key-file=/data/kubernetes/ssl/ca-key.pem \
            --service-account-private-key-file=/data/kubernetes/ssl/ca-key.pem \
            --root-ca-file=/data/kubernetes/ssl/ca.pem \
            --leader-elect=true \
            --v=2 \
            --logtostderr=false \
            --log-dir=/data/kubernetes/logs
Restart=on-failure
LimitNOFILE=65536
[Install]
WantedBy=multi-user.target
```

启动服务

```
systemctl daemon-reload
systemctl start kube-controller-manager
ststemctl status kube-controller-manager
```

## kube-scheduler
kube-scheduler服务文件
```
cat /lib/systemd/system/kube-scheduler.service          
[Unit]
Description=Kubernetes Scheduler Plugin
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
[Service]
ExecStart=/data/kubernetes/bin/kube-scheduler \
 	    --address=127.0.0.1 \
 	    --master=http://127.0.0.1:8080 \
 	    --leader-elect=true \
 	    --v=2 \
            --logtostderr=false \
            --log-dir=/data/kubernetes/logs
Restart=on-failure
LimitNOFILE=65536
[Install]
WantedBy=multi-user.target
```

启动服务

```
systemctl daemon-reload
systemctl start kube-scheduler
ststemctl status kube-scheduler
```
