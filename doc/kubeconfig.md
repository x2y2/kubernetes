## 创建kubeconfig文件

kubelet,kube-proxy在与kube-apiserver进程通信时需要认证和授权，k8s在1.4版后支持kube-apiserver 为客户端生成TLS证书的TLS Bootstrapping功能，这样就不需要为每个客户端生成证书，经过master授权的客户端会自动生成kubeconfig文件和客户端证书。

创建kubeconfig文件需要kubectl命令行工具，先要在master节点上安装kubeclt，然后再进行下面的操作。kubectl工具可以从这里下载https://pan.baidu.com/s/1FEDIPiP-tGBEXNUnepIykA

创建的kubeconfig可以复制到每个node节点复用。

## 创建TLS Bootstrapping Token

以下操作都是在master上进行。

生成bootstrap-toekn.csv
```
export BOOTSTRAP_TOKEN=$(head -c 16 /dev/urandom | od -An -t x | tr -d ' ')
cat > /data/kubernetes/basic-token.csv <<EOF
${BOOTSTRAP_TOKEN},kubelet-bootstrap,10001,"system:kubelet-bootstrap"
EOF
```
创建TLS Bootstrapping kubeconfig
```
cd /data/kubernetes/ssl
#设置集群参数
kubectl config set-cluster kubernetes \
  --certificate-authority=/data/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=https://192.168.100.39:6443 \
  --kubeconfig=bootstrap.kubeconfig
#设置客户端认证参数
kubectl config set-credentials kubelet-bootstrap \
  --toekn=${BOOTSTRAP_TOKEN} \
  --kubeconfig=bootstrap.kubeconfig
#设置上下文参数
kubectl config set-context default \
  --cluster=kubernetes \
  --user=kubelet-bootstrap \
  --kubeconfig=bootstrap.kubeconfig
#设置默认上下文
kubectl confit use-context default --kubeconfig=bootstrap.kubeconfig
```
- 设置客户端认证参数时没有指定证书和密钥，会有kube-apiserver自动生成
##创建TLS kube-proxy kubeconfig

```
cd /data/kubernetes/ssl
#设置集群参数
kubectl config set-cluster kubernetes \
  --certificate-authority=/data/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=https://192.168.100.39:6443 \
  --kubeconfig=kube-proxy.kubeconfig
#设置客户端认证参数
kubectl config set-credentials kube-proxy \
  --client-certificate=/data/kubernetes/ssl/kube-porxy.pem \
  --client-key=/data/kubernetes/ssl/kube-proxy-key.pem
  --kubeconfig=kube-proxy.kubeconfig
#设置上下文参数
kubectl config set-context default \
  --cluster=kubernetes \
  --user=kube-proxy \
  --kubeconfig=kube-proxy.kubeconfig
#设置默认上下文
kubectl confit use-context default --kubeconfig=kube-proxy.kubeconfig
```

## 分发kubeconfig文件

将bootstrap.kubeconfig 和kube-proxy.kubeconfig scp 到各个node节点的/data/kubernetes/ssl
