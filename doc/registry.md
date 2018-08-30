## 私有镜像库配置

下载registry镜像,国内用户可以从daocloud.io下载

docker pull index.docker.io/library/registry

用registry 镜像启动容器

docker run -d --name docker-registry --restart=always -p 5000:5000 -v /data/registry:/var/lib/registry registry

将宿主机的5000端口映射到容器的5000端口，挂载宿主机的/data/registry目录到容器目录/var/lib/registry。默认上传的镜像都是存储在容器的/var/lib/registry目录

- 在所有node节点 echo "192.168.100.34 registry " >> /etc/hosts ，本文是将192.168.100.34作为私钥镜像库
- 所有node节点的docker 配置文件增加--insecure-registry=registry:5000  配置（具体见node.md Docker配置部分）





