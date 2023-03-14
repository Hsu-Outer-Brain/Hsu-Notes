# (1条消息) flink on k8s native 再次实践_Jeseva的博客-CSDN博客
[(1 条消息) flink on k8s native 再次实践\_Jeseva 的博客 - CSDN 博客](https://blog.csdn.net/YouLoveItY/article/details/121957256) 

 基于 flink 1.13.2 版本做的实践  
本次主要实践 flink on k8s [native](https://so.csdn.net/so/search?q=native&spm=1001.2101.3001.7020) 的两种方式, 分别是 sesion 和 application 方式

第一步: k8s 环境准备

```bash
  1, 创建一个namespace
      kubectl create namespace flink-session-cluster-test-1213
  2, 新建一个serviceaccount, 用来提交flink的任务
     kubectl create serviceaccount flink -n flink-session-cluster-test-1213
  3, 做好绑定
     kubectl create clusterrolebinding flink-role-binding-flink-session-cluster-test-1213_flink \
     --clusterrole=edit   --serviceaccount=flink-session-cluster-test-1213:flink     

```

第二步: 镜像准备

```bash
  使用hdfs作为flink的checkpoint存储,所以需要在flink的lib目录中放入hadoop的jar包
  创建Dockerfile文件,并添加如下内容:
vi   Dockerfile
FROM flink:1.13.2-scala_2.11-java8
COPY ./flink-shaded-hadoop-2-uber-2.7.5-10.0.jar $FLINK_HOME/lib/flink-shaded-hadoop-2-uber-2.7.5-10.0.jar

构建image
docker build -t native_realtime:1.0.3 .

后续的session与application均使用该镜像镜像实践

```

为了解决 hosts 映射以及用户自定义 jar 包等问题, 需要使用 yaml 模板

```bash
vi flink-template.yaml

apiVersion: v1
kind: Pod
metadata:
  name: flink-pod-template
spec:
  initContainers:
    - name: artifacts-fetcher
      image: native_realtime:1.0.3
      
      command: ["/bin/sh","-c"]
      args: ["wget http://xxxxxx:8082/flinkhistory/1.13.2/tt.sql -O /opt/flink/usrHome/taa.sql ; wget http://xxxx:8082/flinkhistory/1.13.2/realtime-dw-service-1.0.1-SNAPSHOT.jar -O /opt/flink/usrHome/realtime-dw-service-1.0.1.jar"]
      volumeMounts:
        - mountPath: /opt/flink/usrHome
          name: flink-usr-home
  hostAliases:
  - ip: 10.1.1.103
    hostnames:
    - "cdh103"
  - ip: 10.1.1.104
    hostnames:
    - "cdh104"
  - ip: 10.1.1.105
    hostnames:
    - "cdh105"
  - ip: 10.1.1.106
    hostnames:
    - "cdh106"
  containers:
    
    - name: flink-main-container
      resources:
        requests:
          ephemeral-storage: 2048Mi
        limits:
          ephemeral-storage: 2048Mi
      volumeMounts:
        - mountPath: /opt/flink/usrHome
          name: flink-usr-home
  volumes:
    - name: flink-usr-home
      hostPath:
        path: /tmp
        type: Directory


```

使用 run application 模式提交任务

```
`/data/flink-1.13.0/bin/flink run-application \
    --target kubernetes-application \
	-Dresourcemanager.taskmanager-timeout=345600 \
	-Dkubernetes.namespace=flink-session-cluster-test-1213 \
	-Dkubernetes.service-account=flink \
    -Dkubernetes.cluster-id=flink-stream-reatime-dw11 \
    -Dhigh-availability=org.apache.flink.kubernetes.highavailability.KubernetesHaServicesFactory \
	-Dhigh-availability.storageDir=hdfs://cdh104:8020/flink/recovery \
    -Dkubernetes.container.image=native_realtime:1.0.3 \
	-Dstate.checkpoints.dir=hdfs://cdh104:8020/flink/checkpoints/flink-stream-application-cluster-08 \
    -Dstate.savepoints.dir=hdfs://cdh104:8020/flink/savepoints/flink-stream-application-cluster-08 \
	-Dexecution.checkpointing.interval=2s \
	-Dexecution.checkpointing.mode=EXACTLY_ONCE \
	-Dstate.backend=filesystem \
	-Dkubernetes.rest-service.exposed.type=NodePort  \
	-Drestart-strategy=failure-rate  \
	-Drestart-strategy.failure-rate.delay=1s  \
	-Drestart-strategy.failure-rate.failure-rate-interval=5s \
	-Drestart-strategy.failure-rate.max-failures-per-interval=1  \
	-Dtaskmanager.memory.process.size=1096m \
    -Dkubernetes.taskmanager.cpu=1 \
    -Dtaskmanager.numberOfTaskSlots=1 \
	-Dkubernetes.pod-template-file=./flink-template.yaml \
	-c com.xxx.bigdata.rt.dw.service.runtime.RealtimeWarehouseMain \
    local:///opt/flink/usrHome/realtime-dw-service-1.0.1.jar \
	-cfc state.checkpoint.interval=60000  -cfp 1 -cfm no -cfn kafka_es -cfs /opt/flink/usrHome/taa.sql` 

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-3-14%2018-11-44/9bd840ec-9278-4de9-9c4f-16dc606d70f8.png?raw=true)

*   1
*   2
*   3
*   4
*   5
*   6
*   7
*   8
*   9
*   10
*   11
*   12
*   13
*   14
*   15
*   16
*   17
*   18
*   19
*   20
*   21
*   22
*   23
*   24
*   25
*   26


```

使用[session](https://so.csdn.net/so/search?q=session&spm=1001.2101.3001.7020)模式提交任务

```
`-- 创建session
/data/flink-1.13.0/bin/kubernetes-session.sh \
  -Dkubernetes.cluster-id=stream-wordcount-application-cluster \
  -Dtaskmanager.memory.process.size=1096m \
  -Dkubernetes.taskmanager.cpu=1 \
  -Dtaskmanager.numberOfTaskSlots=4 \
  -Dkubernetes.container.image=native_realtime:1.0.3 \
  -Dkubernetes.service.exposed.type=NodePort \
  -Dkubernetes.jobmanager.service-account=flink \
  -Dkubernetes.service-account=flink \
  -Dkubernetes.namespace=flink-session-cluster-test-1213\
  -Dhigh-availability=org.apache.flink.kubernetes.highavailability.KubernetesHaServicesFactory \
	-Dhigh-availability.storageDir=hdfs://cdh104:8020/flink/recovery \
    -Dkubernetes.container.image=native_realtime:1.0.3 \
	-Dstate.checkpoints.dir=hdfs://cdh104:8020/flink/checkpoints/flink-stream-application-cluster-08 \
    -Dstate.savepoints.dir=hdfs://cdh104:8020/flink/savepoints/flink-stream-application-cluster-08 \
	-Dexecution.checkpointing.interval=2s \
	-Dexecution.checkpointing.mode=EXACTLY_ONCE \
	-Dstate.backend=filesystem \
	-Dkubernetes.rest-service.exposed.type=NodePort  \
	-Drestart-strategy=failure-rate  \
	-Drestart-strategy.failure-rate.delay=1s  \
	-Drestart-strategy.failure-rate.failure-rate-interval=5s \
	-Drestart-strategy.failure-rate.max-failures-per-interval=1  \
	-Dkubernetes.pod-template-file=./flink-template.yaml \

提交任务
/data/flink-1.13.0/bin/flink run -d -e kubernetes-session \
 -Dkubernetes.cluster-id=stream-wordcount-application-cluster \
 -Dkubernetes.namespace=flink-session-cluster-test-1213 \
 -Dkubernetes.taskmanager.service-account=flink \
 -Dexecution.attached=true \
 -c com.xxx.bigdata.rt.dw.service.runtime.RealtimeWarehouseMain \
    ./realtime-dw-service-1.0.1-SNAPSHOT.jar \
	 -cfc state.checkpoint.interval=60000  -cfp 1 -cfm no -cfn kafka_es -cfs ./tt.sql` 

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-3-14%2018-11-44/1fa0e148-02b3-41f6-b18d-a40b352213fd.png?raw=true)

*   1
*   2
*   3
*   4
*   5
*   6
*   7
*   8
*   9
*   10
*   11
*   12
*   13
*   14
*   15
*   16
*   17
*   18
*   19
*   20
*   21
*   22
*   23
*   24
*   25
*   26
*   27
*   28
*   29
*   30
*   31
*   32
*   33
*   34
*   35
*   36


```

问题及解决:  
1, flink 任务的 hosts 问题?  
可以通过 flink 提供的 yaml 模板, 将 hosts 配置放在 yaml 中, 然后在命令使用 - Dkubernetes.pod-template-file 指定  
2, 关于使用 session 模式出现启动 taskManager 时, 获取 configmap 权限不够的问题?  
可以使用 -Dkubernetes.jobmanager.service-account=flink -Dkubernetes.service-account=flink -Dkubernetes.taskmanager.service-account=flink 来解决  
3, 如果使用 application 模式, 解决自定义 jar 包不想打入镜像的问题?  
可以在 yaml 模板中, initContainers 使用 wget 方式引入

session 模式与 application 模式的相互比较  
session 模式, 先启动 jobmanager, 再之后根据提交的任务, 来启动 taskManager, 导致多个任务日志耦合在一起, 但是自定义的 jar 包不需要再构建镜像, 相对提交比较简单  
application 模式, 需要将自定义的 jar 包构建在镜像中, 或者使用 yaml 模板的 initContainers 的方式. 日志可以分开.  
这两种模式均可以实现高可用, 不管是 application 还是 session 模式, 均可以自动以 checkpoint 自动重启

本次未实践日志的收集等相关

追加一下信息, 笔者通过几天的实践, 发现一种更为方便的提交 application 模式的任务, 直接使用 flink 的镜像, 将 hadoop 相关的 jar 包直接放入到 $FLINK_HOME/lib/extr-lib 目录下, flink 会自己加载这个 jar 包

```
`apiVersion: v1
kind: Pod
metadata:
  name: flink-pod-template
spec:
  initContainers:
    - name: artifacts-fetcher
      image: flink:1.13.2-scala_2.11-java8
      
      command: ["/bin/sh","-c"]
	  
      args: ["wget http://xxxx:8082/tt1.sql -O /opt/flink/usrHome/taa.sql ; wget http://xxxx:8082/realtime-dw-service-1.0.1.jar -O /opt/flink/usrHome/realtime-dw-service-1.0.1.jar; wget http://xxxx:8082/udf-1.0.1-SNAPSHOT.jar -O /opt/flink/extr-lib/udf-1.0.1-SNAPSHOT.jar; wget http://xxxx:8082/flink-shaded-hadoop-2-uber-2.7.5-10.0.jar -O /opt/flink/extr-lib/flink-shaded-hadoop-2-uber-2.7.5-10.0.jar"]
      volumeMounts:
        - mountPath: /opt/flink/usrHome
          name: flink-usr-home
        - mountPath: /opt/flink/extr-lib
          name: flink-usr-extr-lib
  hostAliases:
  - ip: 10.1.1.103
    hostnames:
    - "cdh103"
  - ip: 10.1.1.104
    hostnames:
    - "cdh104"
  - ip: 10.1.1.105
    hostnames:
    - "cdh105"
  - ip: 10.1.1.106
    hostnames:
    - "cdh106"
  containers:
    
    - name: flink-main-container
      resources:
        requests:
          ephemeral-storage: 2048Mi
        limits:
          ephemeral-storage: 2048Mi
      volumeMounts:
        - mountPath: /opt/flink/usrHome
          name: flink-usr-home
        
        
        - mountPath: /opt/flink/lib/extr-lib
          name: flink-usr-extr-lib
          
          
  volumes:
    - name: flink-usr-home
      emptyDir: {}
    - name: flink-usr-extr-lib
      emptyDir: {}` 

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-3-14%2018-11-44/9d458cf6-b590-4a50-9fc7-f58aa2dd853e.png?raw=true)

*   1
*   2
*   3
*   4
*   5
*   6
*   7
*   8
*   9
*   10
*   11
*   12
*   13
*   14
*   15
*   16
*   17
*   18
*   19
*   20
*   21
*   22
*   23
*   24
*   25
*   26
*   27
*   28
*   29
*   30
*   31
*   32
*   33
*   34
*   35
*   36
*   37
*   38
*   39
*   40
*   41
*   42
*   43
*   44
*   45
*   46
*   47
*   48
*   49
*   50
*   51
*   52


```

提交命令如下:

```
`/data/flink-1.13.0/bin/flink run-application \
    --target kubernetes-application \
	-Dresourcemanager.taskmanager-timeout=345600 \
	-Dkubernetes.namespace=flink-session-cluster-test-1213 \
	-Dkubernetes.service-account=flink \
    -Dkubernetes.cluster-id=flink-stream-reatime-dw11 \
    -Dhigh-availability=org.apache.flink.kubernetes.highavailability.KubernetesHaServicesFactory \
	-Dhigh-availability.storageDir=hdfs://cdh104:8020/flink/recovery \
    -Dkubernetes.container.image=flink:1.13.2-scala_2.11-java8 \
	-Dstate.checkpoints.dir=hdfs://cdh104:8020/flink/checkpoints/flink-stream-application-cluster-08 \
    -Dstate.savepoints.dir=hdfs://cdh104:8020/flink/savepoints/flink-stream-application-cluster-08 \
	-Dexecution.checkpointing.interval=2s \
	-Dexecution.checkpointing.mode=EXACTLY_ONCE \
	-Dstate.backend=filesystem \
	-Dkubernetes.rest-service.exposed.type=NodePort  \
	-Drestart-strategy=failure-rate  \
	-Drestart-strategy.failure-rate.delay=1s  \
	-Drestart-strategy.failure-rate.failure-rate-interval=5s \
	-Drestart-strategy.failure-rate.max-failures-per-interval=1  \
	-Dtaskmanager.memory.process.size=1096m \
    -Dkubernetes.taskmanager.cpu=1 \
    -Dtaskmanager.numberOfTaskSlots=1 \
	-Dkubernetes.pod-template-file=./flink-template-udf.yaml \
	-c com.xxx.bigdata.rt.dw.service.runtime.RealtimeWarehouseMain \
    local:///opt/flink/usrHome/realtime-dw-service-1.0.1.jar \
	-cfc state.checkpoint.interval=60000  -cfp 1 -cfm no -cfn kafka_es -cfs /opt/flink/usrHome/taa.sql` 

![](https://github.com/Hsu-Outer-Brain/WebCliperCDN_001/blob/main/img3/2023-3-14%2018-11-44/3a434d73-adb7-4dee-83bb-0c4c66134523.png?raw=true)

*   1
*   2
*   3
*   4
*   5
*   6
*   7
*   8
*   9
*   10
*   11
*   12
*   13
*   14
*   15
*   16
*   17
*   18
*   19
*   20
*   21
*   22
*   23
*   24
*   25
*   26


```
