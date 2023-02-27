# (2条消息) PyFlink on K8s 部署实践_bao_since的博客-CSDN博客_pyflink on k8s
[(2 条消息) PyFlink on K8s 部署实践\_bao_since 的博客 - CSDN 博客\_pyflink on k8s](https://blog.csdn.net/bao_since/article/details/113845477) 

## 1.1 Flink 的部署模式

[https://blog.csdn.net/yunxiao6/article/details/108705244](https://blog.csdn.net/yunxiao6/article/details/108705244)

## 1.2 PyFlink on K8s

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-27%2015-51-43/37e41fd3-65f9-4d3e-aac7-b947ac506520.png?raw=true)

**Standalone**：需要配合 kubectl + yaml 部署，Flink 无法感知 K8s 集群的存在，资源被动  
**Native**: 仅使用 flink 客户端 kubernetes-session.sh or flink run 部署，Flink 主动与 K8s 申请资源

## 3.1 K8s 集群

-   K8s >= 1.9 or Minikube
-   KubeConfig （可以查看、创建、删除 pods 和 services）
-   启用 Kubernetes DNS
-   具有 RBAC 权限的 Service Account 可以创建、删除 pods

## 3.2 PyFlink 镜像

```bash
FROM flink:1.12.1-scala_2.11-java8

RUN apt-get update -y && \
      apt-get install -y python3.7 python3-pip python3.7-dev \
      && rm -rf /var/lib/apt/lists/*
RUN rm -rf /usr/bin/python
RUN ln -s /usr/bin/python3 /usr/bin/python

RUN pip3 install apache-flink==1.12.1






RUN mkdir -p $FLINK_HOME/usrlib
COPY /path/of/external/jar/dependencies $FLINK_HOME/usrlib/

COPY /path/of/python/codes /opt/python_codes


```

build PyFlink App image

## 4.1 Session 模式

首先创建静态 session 集群。通过 kubectl create -f yaml 的方式创建 ConfigMap、Service、JobManager Deployment、TaskManager Deployment 资源  
![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-27%2015-51-43/9de99306-5c69-43ac-8e07-9e9af213dcfe.png?raw=true)

向创建好的 session 集群中提交任务。  
![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-27%2015-51-43/bfe377d3-903f-4ee0-a780-11057fca9b96.png?raw=true)

将两个阶段用一张图表示：  
![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-27%2015-51-43/b951450b-50e9-4feb-ba38-1aaebd467ba5.png?raw=true)

-   步骤 1， 使用 Kubectl 或者 K8s 的 Dashboard 提交请求到 K8s Master。
-   步骤 2， K8s Master 将创建 Flink Master Deployment、TaskManager Deployment、ConfigMap、SVC 的请求分发给 Slave 去创建这四个角色，创建完成后 Flink Master、TaskManager 启动。
-   步骤 3， TaskManager 注册到 JobManager。在非 HA 的情况下，是通过内部 Service 注册到 JobManager。
-   至此，Flink 的 Sesion Cluster 已经创建起来。此时就可以提交任务了。
-   步骤 4，在 Flink Cluster 上提交 Flink run 的命令，通过指定 Flink Master 的地址，将相应任务提交上来，用户的 Jar 和 JobGrapth 会在 Flink Client 生成，通过 SVC 传给 Dispatcher。
-   步骤 5，Dispatcher 会发现有一个新的 Job 提交上来，这时会起一个新的 JobMaster，去运行这个 Job。
-   步骤 6，JobMaster 会向 ResourceManager 申请资源，因为 Standalone 方式并不具备主动申请资源的能力，所以这个时候会直接返回，而且我们已经提前把 TaskManager 起好，并且已经注册回来。
-   步骤 7-8，这时 JobMaster 会把 Task 部署到相应的 TaskManager 上，整个任务运行的过程就完成。

### 命令

创建 Session 集群

```bash
kubectl create -f flink-configuration-configmap.yaml
kubectl create -f jobmanager-service.yaml
kubectl create -f jobmanager-session-deployment.yaml
kubectl create -f taskmanager-session-deployment.yaml

kubectl create -f jobmanager-rest-service.yaml

```

提交 Job

```bash

./bin/flink run -m 192.168.0.1:30081 ./examples/streaming/TopSpeedWindowing.jar

sudo flink run -m 192.168.0.1:30081 -pyfs ./examples/python/table/batch -pym word_count

```

回收 Session 集群

```bash
kubectl delete -f jobmanager-rest-service.yaml
kubectl delete -f jobmanager-service.yaml
kubectl delete -f flink-configuration-configmap.yaml
kubectl delete -f taskmanager-session-deployment.yaml
kubectl delete -f jobmanager-session-deployment.yaml

```

## 4.2 Application 模式

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-27%2015-51-43/56d87aaf-c96d-41f6-8dfe-3f7d5d347b09.png?raw=true)

由 Standalone JobCluster EntryPoint 执行，从 classpath 找到用户 Jar，执行它的 main 方法得到 JobGraph 。再提交到 Dispathcher，这时候走 Recover Job 的逻辑，提交到 JobMaster。JobMaster 向 ResourceManager 申请资源，请求 slot，执行 Job。

### 命令

提交 Job

```bash
kubectl create -f flink-configuration-configmap.yaml
kubectl create -f jobmanager-service.yaml
kubectl create -f jobmanager-rest-service.yaml

kubectl create -f jobmanager-application.yaml
kubectl create -f taskmanager-job-deployment.yaml

```

## 4.3 Standalone 部署的不足

-   用户需要对 K8s 有一些最基本的认识，这样才能保证顺利将 Flink 运行到 K8s 之上。
-   Flink 感知不到 K8s 的存在。
-   目前主要使用静态的资源分配。需要提前确认好需要多少个 TaskManager，如果 Job 的并发需要做一些调整，TaskManager 的资源情况必须相应的跟上，否则任务无法正常执行。
-   无法实时申请资源和释放资源。如果维持一个比较大的 Session Cluster，可能会资源浪费。但如果维持的 Session Cluster 比较小，可能会导致 Job 跑得慢或者是跑不起来。

## 5.1 Session 模式

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-27%2015-51-43/ae82e79c-7684-4c3f-b7e5-9e2de5b87660.png?raw=true)

-   第一个阶段：启动 Session Cluster。Flink Client 内置了 K8s Client，告诉 K8s Master 创建 Flink Master Deployment，ConfigMap，SVC。创建完成后，Master 就拉起来了。这时 Session 已经部署完成，没有维护任何 TaskManager。
-   第二个阶段：当用户提交 Job 时，可以通过 Flink Client 或者 Dashboard 的方式，然后通过 Service 到 Dispatcher，Dispatcher 会产生一个 JobMaster。JobMaster 会向 K8sResourceManager 申请资源。ResourceManager 会发现现在没有任何可用的资源，它就会继续向 K8s 的 Master 去请求资源，请求资源之后将其发送回去，起新的 Taskmanager。Taskmanager 起来之后，再注册回来，此时的 ResourceManager 再向它去申请 slot 提供给 JobMaster，最后由 JobMaster 将相应的 Task 部署到 TaskManager 上。这样整个从 Session 的拉起到用户提交都完成了。
-   需注意的是，图中 SVC 是一个外部 Service。必须保证 Flink Client 可以访问到 Jobmanager Dispatcher，否则 Jar 包是无法提交的。

### 命令

创建 Session 集群

```bash
./bin/kubernetes-session.sh \
  -Dkubernetes.cluster-id=session-cluster-1 \
  -Dtaskmanager.numberOfTaskSlots=1 \
  -Dresourcemanager.taskmanager-timeout=3600000 \
  -Dkubernetes.rest-service.exposed.type=NodePort \
  -Dkubernetes.container.image=demo-pyflink-app:1.12.1

```

提交 Job

```bash
./bin/flink run \
    --target kubernetes-session \
    -Dkubernetes.cluster-id=session-cluster-1 \
    -pyfs ./examples/python/table/batch \
    -pym word_count

sudo flink run -m 192.168.0.1:30081 -pyfs ./examples/python/table/batch -pym word_count

```

回收 Session 集群

```bash
 echo 'stop' | ./bin/kubernetes-session.sh \
    -Dkubernetes.cluster-id=session-cluster-1 \
    -Dexecution.attached=true
 
 kubectl delete deployment/session-cluster-1

```

## 5.2 Application 模式

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-2-27%2015-51-43/242568d0-d89b-4ba4-9728-6067e0f2c8f5.png?raw=true)

-   首先创建出了 Service、Master 和 ConfigMap 这几个资源以后，Flink Master Deployment 里面已经带了一个用户 Jar，这个时候 Cluster Entrypoint 就会从用户 Jar 里面去提取出或者运行用户的 main，然后产生 JobGraph。之后再提交到 Dispatcher，由 Dispatcher 去产生 Master，然后再向 ResourceManager 申请资源，后面的逻辑的就和 Session 的方式是一样的。
-   它和 Session 最大的差异就在于它是一步提交的。因为没有了两步提交的需求，如果不需要在任务起来以后访问外部 UI，就可以不用外部的 Service。可直接通过一步提交使任务运行。通过本地的 port-forward 或者是用 K8s ApiServer 的一些 proxy 可以访问 Flink 的 Web UI。此时，External Service 就不需要了，意味着不需要再占用一个 LoadBalancer 或者占用 NodePort。

### 命令

提交 Job

```bash

./bin/flink run-application -p 2 -t kubernetes-application \
  -Dkubernetes.cluster-id=app-cluster \
  -Dtaskmanager.memory.process.size=4096m \
  -Dkubernetes.taskmanager.cpu=2 \
  -Dtaskmanager.numberOfTaskSlots=4 \
  -Dkubernetes.container.image=demo-pyflink-app:1.12.1 \
  -pyfs /opt/python_codes \
  -pym new_word_count
 

./bin/flink run-application -p 2 -t kubernetes-application \
        -Dexecution.attached=true \
        -Dkubernetes.cluster-id=app-cluster \
        -Dtaskmanager.memory.process.size=4096m \
        -Dkubernetes.taskmanager.cpu=2 \
        -Dtaskmanager.numberOfTaskSlots=4 \
        -Dkubernetes.container.image=demo-pyflink-app:1.11.3 \
        -Dpython.files=/opt/python_codes \
        -c org.apache.flink.client.python.PythonDriver local:///opt/flink/opt/flink-python_2.11-1.11.2.jar \
        -py /opt/python_codes/PyFlinkDriver.py

```
