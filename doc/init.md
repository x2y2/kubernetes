
######################
wangpei456@126.com
######################

## 环境说明

```
master 192.168.100.39
registry 192.168.100.34
node1 etcd1 192.168.100.42
node2 etcd2 192.168.100.47
node3 etcd3 192.168.100.48

OS: Ubuntu 16.04

root@master:~# kubectl get nodes
NAME      STATUS    ROLES     AGE       VERSION
node1     Ready     <none>    3d        v1.10.0
node2     Ready     <none>    3d        v1.10.0
node3     Ready     <none>    3d        v1.10.0
node4     Ready     <none>    6h        v1.10.0
```

## 目录结构说明

k8s的家目录在/data/kubernetes，在家目录下有conf,ssl,logs,addons,bin目录
```
conf 配置文件目录
ssl 证书和私钥目录
logs 日志目录
bin 可执行文件目录
addons yaml文件目录
```
bin 目录下的文件需要建软连到/usr/local/bin下
```
ln -s /data/kubernetes/bin/* /usr/local/bin
```
