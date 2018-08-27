## 制作证书
利用cfssl制作证书，在master上操作，制作好的证书分发到node节点使用
```
mkdir /etc/kubernetes/ssl
cd /etc/kubernetes/ssl
wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64
mv cfssl_linux-amd64 cfssl
mv cfssljson_linux-amd64 cfssljson
mv cfssl-certinfo_linux-amd64 cfssl-certinfo
chmod +x cfssl*
```
创建生成CA文件的json配置文件模板和签名请求的模板
```
cfssl print-defaults config > config.json
cfssl print-defaults csr > csr.json
```
建立CA的json文件
```
vim ca-config.json
{
  "signing": {
    "default": {
      "expiry": "87600h"
    },
    "profiles": {
      "kubernetes": {
        "usages": [
            "signing",
            "key encipherment",
            "server auth",
            "client auth"
        ],
        "expiry": "87600h"
      }
    }
  }
}
```

建立CA的签名请求文件

```
vim ca-csr.json
{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "ShangHai",
      "L": "ShangHai",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
```

生成CA证书和私钥
```
cfssl gencert -initca ca-csr.json | cfssljson -bare ca
ls ca*
ca-config.json  ca.csr  ca-csr.json  ca-key.pem  ca.pem
```
创建kubernetes的csr文件

```
vim kubernetes-csr.json 
{
    "CN": "kubernetes",
    "hosts": [
       "127.0.0.1",
       "10.254.0.1",
       "192.168.100.39",
       "192.168.100.42",
       "192.168.100.47",
       "192.168.100.48",
       "kubernetes",
       "kubernetes.default",
       "kubernetes.default.svc",
       "kubernetes.default.svc.cluster",
       "kubernetes.default.svc.cluster.local"
     ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "ShangHai",
            "ST": "ShangHai",
            "O": "k8s",
            "OU": "System"
        }
    ]
}
```

hosts 字段不为空则需要指定授权使用该证书的 IP 或域名列表,该证书将会被etcd集群和kubernetes master使用到，所以hosts列表中需要指定etc集群和kubernetes master的IP和kubernetes 服务的IP

创建kubernetes证书和私钥
```
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kubernetes-csr
.json |cfssljson -bare kubernetes
ls kubernetes*
kubernetes.csr  kubernetes-csr.json  kubernetes-key.pem  kubernetes.pem
```
生成admin的csr文件

```
vim admin-csr.json 
{
    "CN": "admin",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "ShangHai",
            "ST": "ShangHai",
            "O": "system:masters",
            "OU": "System"
        }
    ]
}
```

kube-apiserver 预定义了一些 RBAC 使用的 RoleBindings，如 cluster-admin 将 Group system:masters 与 Role cluster-admin 绑定，该 Role 授予了调用kube-apiserver 的所有 API的权限

O 指定该证书的 Group 为 system:masters，kubelet 使用该证书访问 kube-apiserver 时 ，由于证书被 CA 签名，所以认证通过，同时由于证书用户组为经过预授权的 system:masters，所以被授予访问所有 API 的权限

这个admin 证书，是将来生成管理员用的kube config 配置文件用的，现在我们一般建议使用RBAC 来对kubernetes 进行角色权限控制， kubernetes 将证书中的CN 字段 作为User， O 字段作为 Group

```
root@master:~# kubectl get clusterrolebinding cluster-admin  -o yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  creationTimestamp: 2018-08-24T14:27:33Z
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: cluster-admin
  resourceVersion: "92"
  selfLink: /apis/rbac.authorization.k8s.io/v1/clusterrolebindings/cluster-admin
  uid: d701ac5e-a7a9-11e8-8c0d-5254001f7ab1
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: system:masters
```

创建admin证书和私钥

cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes admin-csr.json
 |cfssljson -bare admin

生成kube-proxy 的csr文件

```
vim kube-proxy-csr.json 

{
    "CN": "system:kube-proxy",
    "hosts": [],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "ShangHai",
            "ST": "ShangHai",
            "O": "k8s",
            "OU": "System"
        }
    ]
}
```

CN 指定该证书的 User 为 system:kube-proxy

kube-apiserver 预定义的 ClusterRoleBinding system:node-proxier 将User system:kube-proxy 与 Role system:node-proxier 绑定，该 Role 授予了调用 kube-apiserver Proxy 相关 API 的权限

```
root@master:~# kubectl get clusterrolebinding system:node-proxier  -o yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  creationTimestamp: 2018-08-24T14:27:33Z
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:node-proxier
  resourceVersion: "95"
  selfLink: /apis/rbac.authorization.k8s.io/v1/clusterrolebindings/system%3Anode-proxier
  uid: d7263aad-a7a9-11e8-8c0d-5254001f7ab1
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:node-proxier
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: User
  name: system:kube-proxy
```

生成kube-proxy证书和私钥

cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-proxy-csr
.json |cfssljson -bare kube-proxy



生成的 CA 证书和秘钥文件如下：

```
ca-key.pem
ca.pem
kubernetes-key.pem
kubernetes.pem
kube-proxy.pem
kube-proxy-key.pem
admin.pem
admin-key.pem
```
使用证书的组件如下：
```
etcd：使用 ca.pem、kubernetes-key.pem、kubernetes.pem；
kube-apiserver：使用 ca.pem、kubernetes-key.pem、kubernetes.pem；
kubelet：使用 ca.pem；
kube-proxy：使用 ca.pem、kube-proxy-key.pem、kube-proxy.pem；
kubectl：使用 ca.pem、admin-key.pem、admin.pem；
kube-controller-manager：使用 ca-key.pem、ca.pem
```
创建证书都在master上创建，并且创建一次就可以了，以后需要添加节点就直接复制一份过去就可以用

证书分发
scp 对应的证书到对应的节点即可。
