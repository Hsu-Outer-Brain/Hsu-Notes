# (3条消息) Macbook Pro搭建K8s环境_mac本上搭建k8s_cogo_Time的博客-CSDN博客
[(3 条消息) Macbook Pro 搭建 K8s 环境\_mac 本上搭建 k8s_cogo_Time 的博客 - CSDN 博客](https://blog.csdn.net/cogo_Time/article/details/117446695) 

## 背景说明

在 macos 上安装[docker](https://so.csdn.net/so/search?q=docker&spm=1001.2101.3001.7020)，发现 kubernetes 一直处于 starting 状态，无法启动。在网上也找了些资料来解决，问题一直没有得到解决，因此将自己的解决方式记录下来，以帮助更多的人。

## 版本信息

mac os 版本: macos catalina 10.15.7  
docker 版本：docker desktop community 19.03.8  
[kubernetes](https://so.csdn.net/so/search?q=kubernetes&spm=1001.2101.3001.7020)版本：kubernetes v1.15.5

## 处理步骤

1\. 卸载重装 Docker  
docker -> preferences -> troubleshoot -> uninstall 卸载 docker desktop  
![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-3-8%2016-13-21/ca6f7b59-2c8b-4e91-8fc7-b144216eaadd.png?raw=true)

2\. 新建拉取镜像脚本 docker-images-k8s.sh

```bash
#!/bin/bash
set -e
KUBE_VERSION=v1.15.5
KUBE_DASHBOARD_VERSION=v1.10.1
KUBE_PAUSE_VERSION=3.1
ETCD_VERSION=3.3.10
COREDNS_VERSION=1.3.1

GCR_URL=k8s.gcr.io  

ALIYUN_URL=registry.cn-hangzhou.aliyuncs.com/google_containers

images=(kube-proxy:${KUBE_VERSION}
kube-scheduler:${KUBE_VERSION}
kube-controller-manager:${KUBE_VERSION}
kube-apiserver:${KUBE_VERSION}
pause:${KUBE_PAUSE_VERSION}
etcd:${ETCD_VERSION}
coredns:${COREDNS_VERSION}
kubernetes-dashboard-amd64:${KUBE_DASHBOARD_VERSION})
for imageName in ${images[@]} ; do
docker pull $ALIYUN_URL/$imageName
docker tag $ALIYUN_URL/$imageName $GCR_URL/$imageName
docker rmi $ALIYUN_URL/$imageName
done
docker images

```

3\. 拉取镜像

```bash

chmod 755 docker-images-k8s.sh

./docker-images-k8s.sh 

```

4\. 修改 daocke 镜像标签

```bash
docker tag da86e6ba6ca1 k8s.gcr.io/pause:3.1
docker tag 1399a72fa1a9 k8s.gcr.io/kube-controller-manager:v1.15.5
docker tag fab2dded59dd k8s.gcr.io/kube-scheduler:v1.15.5
docker tag cbd7f21fec99 k8s.gcr.io/kube-proxy:v1.15.5
docker tag e534b1952a0d k8s.gcr.io/kube-apiserver:v1.15.5
docker tag 2c4adeb21b4f k8s.gcr.io/etcd:3.3.10
docker tag eb516548c180 k8s.gcr.io/coredns:1.3.1
docker tag f9aed6605b81 k8s.gcr.io/kubernetes-dashboard-amd64:v1.10.1

```

5\. 勾选 k8s 选项，重启 Docker  
![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-3-8%2016-13-21/9632158e-1d3c-4c23-8573-77806c0fcb79.png?raw=true)

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-3-8%2016-13-21/daef2a02-7e4f-48b1-98b9-c43b0c7fec5f.png?raw=true)

> 如果是安装新版本的 k8s，请参考：[https://www.jianshu.com/p/e5c056baa8ab](https://www.jianshu.com/p/e5c056baa8ab)
