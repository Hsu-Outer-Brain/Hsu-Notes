# 网站项目部署在k8s案例与Jenkins自动化发布项目（CI/CD）
[网站项目部署在 k8s 案例与 Jenkins 自动化发布项目（CI/CD）](http://e.betheme.net/article/show-1257101.html?action=onClick) 

 2023/5/16 15:35:33

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-5-16%2015-55-01/038c4865-2233-4c70-bbb6-f901d5751014.png?raw=true)

## 在 K8s 平台部署 Java 网站项目

## 制作镜像流程

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-5-16%2015-55-01/37f7bc65-2c14-47e3-bc7c-6e25014c9002.png?raw=true)

### 第一步：制作镜像

```null
使用镜像仓库（私有仓库、公共仓库）：
1、配置可信任（如果仓库是HTTPS访问不用配置）
# vi /etc/docker/daemon.json
{
"insecure-registries": ["192.168.31.90"]
}
2、 将镜像仓库认证凭据保存在K8s Secret中
kubectl create secret docker-registry registry-auth \
--docker-username=admin \
--docker-password=Harbor12345 \
--docker-server=192.168.31.90
3、在yaml中使用这个认证凭据
imagePullSecrets:
- name: registry-auth
```

### 第二步：使用控制器部署镜像

模板

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-5-16%2015-55-01/994f2c62-71fa-4324-b50b-9ffbe37d0288.png?raw=true)

Pod 主要配置启动容器属性：

> • 变量
>
> • 资源配额
>
> • 健康检查
>
> • 卷挂载点

案例：

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-5-16%2015-55-01/1b7e9c50-d6e3-4f09-b478-5d54b147474f.png?raw=true)

### 第三步：部署数据库

1、使用 deployment 部署一个 mysql 实例， service 暴露访

```null
kubectl apply -f mysql.yaml
kubectl get pod,svc
```

2、测试 mysql 实例是否可以访问

```null
kubectl run mysql-client --rm -it --image=mysql:5.7.30 – bash
/# mysql -h10.106.166.31 -uroot -p'123456' #10.106.166.31为mysql ClusterIP
mysql> show databases;
```

3、导入项目 sql 文件

```null
kubectl cp db/tables_ly_tomcat.sql mysql-client:/ # 将sql文件拷贝到mysql客户端容器中
/# mysql -h10.106.166.31 -uroot -p'123456'
mysql> create database test;
mysql> use test;
mysql> source /tables_ly_tomcat.sql;
mysql> show tables; # 只有一个user表
```

### 第四步：对外暴露应用

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-5-16%2015-55-01/4bd6f58c-3e2e-4d16-b70f-46acd0be55a0.png?raw=true)

### 第五步：增加公网负载均衡器

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-5-16%2015-55-01/07c0c5bc-4922-4a62-b195-2456271c0215.png?raw=true)

* * *

## 发布流程设计

![](https://img-blog.csdnimg.cn/e5cd3bd0cfef46078905bae4a6e8ac83.png)

使用 Gitlab 作为代码仓库 & 使用 Harbor 作为镜像仓库

### Harbor 镜像仓库

```null
项目地址： https://github.com/goharbor/harbor
```

### 部署步骤：

```null
# tar zxvf harbor-offline-installer-v2.0.0.tgz
# cd harbor
# cp harbor.yml.tmpl harbor.yml# vi harbor.ymlhostname: 192.168.31.90
https: # 先注释https相关配置
harbor_admin_password: Harbor12345# ./prepare
# ./install.sh
# docker-compose ps
```

### Gitlab 代码仓库

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-5-16%2015-55-01/9301698e-6d25-42dd-bd95-eb3921184bca.png?raw=true)

> Jenkins 是一款开源 CI&CD 系统，用于自动化各种任务，包括构建、测试和部署。
>
> Jenkins 官方提供了镜像： Docker 使用 Deployment 来部署这个镜像，会暴露两个端口： 8080 Web 访问端口， 50000 Slave 通信端口，容器启动后 Jenkins 数据存储在 / var/jenkins_home 目录，所以需要将该目录使用 PV 持久化存储。

## 先安装后面所需的插件：

Jenkins 下载插件默认服务器在国外，会比较慢，建议修改国内源：

\# 进入到 nfs 共享目录

```null
cd /ifs/kubernetes/ops-jenkins-pvc-xxx
sed -i 's/https:\/\/updates.jenkins.io\/download/https:\/\/mirrors.tuna.tsinghua.edu.cn\/jenkins/g' default.json && \
sed -i 's/http:\/\/www.google.com/https:\/\/www.baidu.com/g' default.json
```

### # 重建 Jenkins

```null
http://NodeIP:30008/restart
```

管理 Jenkins-> 系统配置 --> 管理插件 --> 分别搜索 Git Parameter/Git/Pipeline/kubernetes/Config File Provider，选中点击安装。

> • Git：拉取代码
>
> • Git Parameter： Git 参数化构建
>
> • Pipeline：流水线
>
> • kubernetes：连接 Kubernetes 动态创建 Slave 代理
>
> • Config File Provider： 存储配置文件
>
> • Extended Choice Parameter：扩展选择框参数，支持多选

## Jenkins 主从架构

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-5-16%2015-55-01/1f08620c-b575-47ca-a59d-8b03837c9c9f.png?raw=true)

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-5-16%2015-55-01/e16899bb-8705-4c2f-b96a-a7c211bdb3b4.png?raw=true)

当触发 Jenkins 任务时， Jenkins 会调用 Kubernetes API 创建 Slave Pod， Pod 启动后会连接 Jenkins，接受任务并处理。

## Kubernetes 插件配置

Kubernetes 插件： 用于 Jenkins 在 Kubernetes 集群中运行动态代理

插件介绍： [https://github.com/jenkinsci/kubernetes-plugin](https://github.com/jenkinsci/kubernetes-plugin)

配置插件： 管理 Jenkins-> 管理 Nodes 和云 -> 管理云 -> 添加 Kubernetes

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-5-16%2015-55-01/e28623d2-ed90-496e-b66e-8539629e767b.png?raw=true)

## 自定义 Jenkins-Slave 镜像

### 构建 Slave 镜像 Dockerfile（结合项目环境）

```null
FROM centos:7
LABEL maintainer lizhenliang
RUN yum install -y java-1.8.0-openjdk maven git libtool-ltdl-devel && \
yum clean all && \
rm -rf /var/cache/yum/* && \
mkdir -p /usr/share/jenkins
COPY slave.jar /usr/share/jenkins/slave.jar
COPY jenkins-slave /usr/bin/jenkins-slave
COPY settings.xml /etc/maven/settings.xml
RUN chmod +x /usr/bin/jenkins-slave
COPY kubectl /usr/bin/
ENTRYPOINT ["jenkins-slave"]
```

![](https://img-blog.csdnimg.cn/2bbf650d0267404ca4291ab55584bad8.png)

![](https://img-blog.csdnimg.cn/10ce752c986046aa87653006ca805635.png)

## 测试主从架构是否正常

![](https://img-blog.csdnimg.cn/da35641a51ba46d486c8a18283fc548e.png)

## Jenkins Pipeline（流水线）

Jenkins Pipeline 是一套运行工作流框架，将原本独立运行单个或者多个节点的任务链接起来，实现单个任务难以完成的复杂流程编排和可视化。

> • Jenkins Pipeline 是一套插件，支持在 Jenkins 中实现持续集成和持续交付；
>
> • Pipeline 通过特定语法对简单到复杂的传输管道进行建模；
>
> • Jenkins Pipeline 的定义被写入一个文本文件，称为 Jenkinsfile。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-5-16%2015-55-01/92fe53e5-6145-4e0d-b6e7-b7fca006f70c.png?raw=true)

## Jenkins Pipeline 语法

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-5-16%2015-55-01/97793abc-d390-4b1f-9d3d-530a5279d010.png?raw=true)

Stages 是 Pipeline 中最主要的组成部分， Jenkins 将会按照 Stages 中描述的顺序从上往下的执行。

> • Stage：阶段，一个 Pipeline 可以划分为若干个 Stage，每个 Stage 代表一组操作，
>
> 比如： Build、 Test、 Deploy
>
> • Steps：步骤， Steps 是最基本的操作单元，可以是打印一句话，也可以是构建一个 Docker 镜像，由各类 Jenkins 插件提供，比如命令： sh ‘mvn'，就相当于我们平时 shell 终端中执行 mvn 命令一样。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-5-16%2015-55-01/f577e54d-669a-4442-bf65-aa65eaa3fee8.png?raw=true)

## Jenkins 流水线自动发布项目

### 思路 - 项目部署流程

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-5-16%2015-55-01/1f6c88b0-d554-485a-9a5c-52bc3541f7bd.png?raw=true)

在实际工作中，会维护多个项目，如果每个服务都创建一个 item，势必给运维维护成本增加很大， 因此需要编写一个通用 Pipeline 脚本，将这些项目部署差异化部分使用 Jenkins 参数化，人工交互确认发布的分支、副本数、命名空间等。

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-5-16%2015-55-01/1fe50c18-2645-42c2-b62f-4b742fbfb53f.png?raw=true)

> 将部署项目 yaml 文件提交到项目代码仓库里，在 Slave 容器里使用 kubectl apply 部署。
>
> 由于 kubectl 使用 kubeconfig 配置文件连接 k8s 集群，还需要通过 Config File Provider 插件将 kubeconfig 配置文件存储到 Jenkins，然后再挂载到 Slave 容器中， 这样就有权限部署了（kubectl apply deploy.yaml --kubeconfig=config）

注：为提高安全性， kubeconfig 文件应分配权限

除了上述方式，还可以使用 Kubernetes Continuous Deploy 插件， 将资源配置（YAML） 部署到 Kubernetes，这种不

是很灵活性

## 流水线脚本与源代码一起版本管理

Jenkinsfile 文件建议与源代码一起版本管理，实现流水线即代码（Pipeline as Code）。

这样做的好处：

• 自动为所有分支创建流水线脚本

• 方便流水线代码复查、追踪、迭代

• 可被项目成员查看和编辑

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-5-16%2015-55-01/3d80d0e0-2922-4a36-9275-ad46df21822c.png?raw=true)

## Jenkins 从 Git 仓库中读取 Jenkinsfile

### k8s 容器云平台架构

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-5-16%2015-55-01/c2206093-706c-4bbb-bfda-bc58174a8424.png?raw=true)
