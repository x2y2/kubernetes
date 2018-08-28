##目录结构说明

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
