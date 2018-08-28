##创建kubeconfig文件

kubelet,kube-proxy在与kube-apiserver进程通信时需要认证和授权，k8s在1.4版后支持kube-apiserver 为客户端生成TLS证书的TLS Bootstrapping功能，这样就不需要为每个客户端生成证书，经过master授权的客户端会自动生成kubeconfig文件和客户端证书。

创建kubeconfig文件需要kubectl命令行工具，先要在master节点上安装kubeclt，然后再进行下面的操作。kubectl工具可以从这里下载https://pan.baidu.com/s/1FEDIPiP-tGBEXNUnepIykA

创建的kubeconfig可以复制到每个node节点复用。

##创建TLS Bootstrapping Token

以下操作都是在master上进行。

生成bootstrap-toekn.csv
```
export BOOTSTRAP_TOKEN=$(head -c 16 /dev/urandom | od -An -t x | tr -d ' ')
cat > /data/kubernetes/basic-token.csv <<EOF
${BOOTSTRAP_TOKEN},kubelet-bootstrap,10001,"system:kubelet-bootstrap"
EOF
```
