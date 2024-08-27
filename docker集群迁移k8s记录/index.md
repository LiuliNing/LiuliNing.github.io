# Docker集群迁移K8s记录


<!--more-->
<!--more-->

# 总体流程学习和记录
```
1、K8s学习，研发环境搭建
2、评估和规划迁移资源的范围
3、整理当前集群所使用的配置信息和依赖卷信息以及在k8s中如何配置
4、使用Kompose工具Docker Compose文件转换为K8s配置文件（如Deployment、Service、ConfigMap、Secret、Pod、Volume等）
5、研发环境打包脚本，流水线自动化到k8s
6、腾讯云TKE、TSF组件学习
7、云服务资源申请，网络申请，迁移云服务后的测试、性能监控与调试
```

## 一、K8s学习

### 官网文档  

https://kubernetes.io/zh-cn/docs/concepts/security/secrets-good-practices/

博客

https://jimmysong.io/

### 目标

```
安装 kubernetes 集群。包括 minikube，云平台搭建，裸机搭建  
部署项目到集群中，对外暴露服务端口  
部署数据库这种有状态的应用，数据持久化  
集群中配置文件和密码文件
使用 Helm 应用商店快速安装第三方应用  
Ingress 对外提供服务
```


### 差异对比
```
传统部署方式：  
应用直接在物理机上部署，机器资源分配不好控制，出现Bug时，可能机器的大部分资源被某个应用占用，导致其他应用无法正常运行，无法做到应用隔离。  
虚拟机部署  
在单个物理机上运行多个虚拟机，每个虚拟机都是完整独立的系统，性能损耗大。  
容器部署  
所有容器共享主机的系统，轻量级的虚拟机，性能损耗小，资源隔离，CPU和内存可按需分配
```


### 关键概念
```

master  
主节点，控制平台，不需要很高性能，不跑任务，通常一个就行了，也可以开多个主节点来提高集群可用度。  
worker  
工作节点，可以是虚拟机或物理计算机，任务都在这里跑，机器性能需要好点；通常都有很多个，可以不断加机器扩大集群；每个工作节点由主节点管理  
Pod  
K8S 调度、管理的最小单位，一个 Pod 可以包含一个或多个容器，每个 Pod 有自己的虚拟IP。一个工作节点可以有多个 pod，主节点会考量负载自动调度 pod 到哪个节点运行。

### Kubernetes 组件
kube-apiserver API 服务器，公开了 Kubernetes API  
etcd 键值数据库，可以作为保存 Kubernetes 所有集群数据的后台数据库  
kube-scheduler 调度 Pod 到哪个节点运行  
kube-controller 集群控制器  
cloud-controller 与云服务商交互
namespace
pvc
service
pod
StatefulSet 持久化配置
Deployment
configmap
```



### Docker镜像管理 

```
docker Registry  Harbor
harbor离线搭建，本地登录
	离线安装
		使用官方离线包，解压后一键进行安装
			离线镜像导入和同步
				单个镜像导入
				离线框架多镜像批量导入
					获取当前容器中所有离线sinfcloud镜像
						dk images | grep sinfcloud | awk '{print $1}'
					脚本批量循环导入
	在线安装
		在线镜像同步，远程库-本地库
			配置远程仓库的同步复制，指定标签、版本等信息
```



### 文件系统 NFS系统  

```
主节点搭建NFS，存储共享文件，安装包及配置文件，子节点挂载目录
```



### 异常处理
	yaml文件写错
		kube-linter这个工具来检查yaml语法是否有误
	系统版本与脚本不匹配
	节点ip变化
		主节点
		node
		如何保障ip变化不影响集群运行
	ndf客户端挂载报错
		mount: 文件系统类型错误、选项错误  上有坏超级块......
			将节点机器安装nfs客户端依赖，再进行连接测试
		写错挂载地址，且未使用软连接，导致挂载目录无法df -h 、ll 等命令进行查看
			强行取消挂载点
				umount -fl /opt/nfs_data
		nfs实现共享目录对于集群高可用风险，nfs客户端容易卡死
			客户端nfs中有一个内核级别的线程，nfsv4.1-svc，该线程会一直和nfs服务端进行通信，且无法被kill掉。（停止客户端Nfs服务，设置开机不自启动，并卸载nfs，重启主机才能让该线程停掉）。
			mount -t nfs  -o rw,intr,soft,timeo=30,retry=3 nfs-server://share-path local-path
	
	主节点ip变化
		固定虚拟机ip，进行端口转发配置
	harbor
		离线导入失败
			镜像命名需要与harbor的规则一致
		导入镜像时区不正确
### 语句
	k8s
		删除指定名称的子节点
			kubectl delete node <nodeName>
		配置主机hostname
			hostnamectl set-hostname 
		kubernetes调度pod运行于master节点上
			让 master节点恢复不参与POD负载的命令为
				kubectl taint nodes <node-name> node-role.kubernetes.io/master=:NoSchedule
			让 master节点恢复不参与POD负载，并将Node上已经存在的Pod驱逐出去的命令
				kubectl taint nodes <node-name> node-role.kubernetes.io/master=:NoExecute
		子节点加入主节点
			kubeadm join <master‐ip>:6443 ‐‐token <kubernetes‐token> --discovery-tokenunsafe-skip-ca-verification
		脚本执行
			子节点安装并加入主节点
				chmod +x install.sh && ./install.sh join <master‐ip>:6443 ‐‐token <kubernetes‐token>
			安装主节点
				chmod +x install.sh && ./install.sh master
	docker
		离线安装compose
			复制离线文件、授权
				sudo chmod +x /usr/local/bin/docker-compose
		常用命令
			重新加载配置文件并重启
				systemctl daemon-reload && systemctl restart docker
			查看包含某个名字的镜像信息，查看列
				dk images | grep sinfcloud | awk '{print $1}'

## 二、迁移准备和环境布置

因为所有服务已经容器化，基本上无需特殊处理，修改部署脚本后进行上传测试即可

以下流程整理自各种博客和chatgpt的提示，基本上迁移的步骤也是按照这个来的

### 清理Docker镜像

审查并清理Docker镜像，删除不必要的组件和文件，确保镜像精简且包含应用程序的所有必要部分。

###  配置环境变量

将Docker容器中使用的环境变量和配置文件迁移到Kubernetes中的ConfigMaps和Secrets中，以确保在Kubernetes中正确设置运行时变量。

### 创建Deployment

使用Kubernetes的Deployment资源来定义应用程序的期望状态和副本数量。Deployment负责在集群中创建Pod，并确保它们按照定义的规则进行运行。

### 创建Service

使用Service资源定义将外部流量引导到应用程序的方式。Service提供了一个稳定的网络端点，使得应用程序可以被集群内和集群外的其他组件访问。

###  添加健康检查

Kubernetes通过健康检查确定Pod的运行状况。确保在Docker容器中添加适当的健康检查，以便Kubernetes可以正确监控和管理Pod的状态。

### 采用Kubernetes标准

遵循Kubernetes的最佳实践，例如使用标签（Labels）和注释（Annotations）来描述和组织应用程序。

### 使用PV/PVC管理持久化数据

容器中的存储都是临时的，因此Pod重启的时候，内部的数据会发生丢失。实际应用中，我们有些应用是无状态，有些应用则需要保持状态数据，确保Pod重启之后能够读取到之前的状态数据，有些应用则作为集群提供服务。这三种服务归纳为无状态服务、有状态服务以及有状态的集群服务，其中后面两个存在数据保存与共享的需求，因此就要采用容器外的存储方案。

### 使用ConfigMap管理应用配置文件

在DevOps的部署流水线中，我们强调代码和配置的分离，这样更容易实现流水线的编排。在Kubernetes中提供了ConfigMap资源对象，其实ConfigMap和Secret都是一种卷类型，可以从文件、文件夹等途径创建ConfigMap。然后再Pod中挂载使用。

### 实施滚动更新和回滚策略

#### 滚动更新

使用Deployment的滚动更新功能，逐步将新版本的Pod引入集群，确保应用程序在升级过程中保持稳定。

#### 回滚策略

如果升级后发现问题，可以使用Deployment的回滚策略，将应用程序回滚到之前的稳定版本。

### 资源定义

Kubernetes使用资源定义（Resource Requests）和资源限制（Resource Limits）来控制Pod对集群资源的使用。Requests定义了Pod所需资源的最小量，而Limits定义了Pod在超出此限制时可能被终止的资源量。

### 节点选择和亲和性

通过Node选择器和亲和性规则，可以将Pod调度到特定的节点上，以满足应用程序对硬件特性或数据存储的需求，从而实现更灵活的资源管理。

### 横向Pod自动伸缩

使用Horizontal Pod Autoscaler（HPA）自动根据CPU或内存的使用情况调整Pod的副本数量。HPA可确保在高负载时增加实例，在低负载时减少实例，以有效利用集群资源。

### Pod的水平扩展

Kubernetes允许手动调整Deployment的副本数量，实现Pod的水平扩展。通过以下命令可以增加或减少Pod的实例数量。

### 自动缩放

利用Kubernetes的自动缩放机制，集群可以根据负载的变化自动增加或减少节点的数量。这样可以确保在高负载时有足够的资源，并在低负载时减少资源浪费。

### 网络通信

在Kubernetes环境中，确保容器之间的安全网络通信至关重要。本节将介绍如何实现容器间的网络隔离以及使用Kubernetes Network Policies加强安全性。

### 容器间的网络隔离

Kubernetes使用命名空间（Namespace）来提供虚拟的集群，可以将不同的资源隔离开。容器在同一命名空间中共享网络命名空间，但可以通过使用不同的命名空间来实现网络隔离。

### Pod之间的通信

通过Service资源，Kubernetes提供了一种逻辑上的抽象，将一组Pod封装在一个虚拟的服务IP地址下。这种方式实现了容器间的通信，同时隐藏了底层Pod的IP地址变化。

### 监控和管理

使用开源监控工具（如Prometheus、Grafana）或云服务提供商的监控服务，设置应用程序的监控系统。监控关键指标，如CPU使用率、内存消耗、请求响应时间等。

### 日志收集

使用日志收集工具（如ELK Stack、Fluentd）或云服务提供商的日志服务，集中管理应用程序生成的日志。通过搜索和分析日志，可以更容易地诊断问题并进行性能优化。

### 配置CI/CD流水线

选择适用于Kubernetes的CI/CD工具（如Jenkins、GitLab CI、CircleCI），配置流水线以自动构建Docker镜像、运行测试，并将应用程序部署到Kubernetes集群。



### 其他：

### 时区的配置问题

从官方下载的镜像都会有默认时区，一般我们使用的时候都需要更改时区，更改时区的方式有多种，这里简单说两种。一是将容器镜像的/etc/loacltime根据需要设置为对应的时区，二是采用配置文件中的volume挂载宿主机对应的localtime文件的方式。推荐采用第二种方式。

### 使用 Helm 

使用 Helm 管理所有的 资源对象的定义（yaml文件）；

### Service 的命名 

一般是 “业务名-应用服务器类型-其他标识”

### NameSpace的区分

集群内部同时运行开发、测试、staging、生产环境，通过NameSpace实现不同运行环境的隔离，同时应用软件在不同的运行环境之间也不会产生命名冲突。

## 三、脚本记录

### Namespace

```
apiVersion: v1
kind: Namespace
metadata:
  name: sinfcloud
```

### Pvc

```
############################
####### 默认nfs为存储 ########
############################
kind: PersistentVolume
apiVersion: v1
metadata:
  name: sc-pv-sinfcloud-data
  namespace: sinfcloud
  annotations:
    pv.kubernetes.io/provisioned-by: cluster.local/nfs-client-nfs-client-provisioner
  finalizers:
    - kubernetes.io/pv-protection
spec:
  capacity:
    storage: 10Gi
  nfs:
    server: 192.168.1.187
    path: /home/nfs #确保当前路径存在
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete
  storageClassName: nfs-client
  volumeMode: Filesystem
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: sincloud-pvc
  namespace: sinfcloud
  annotations:
    kubesphere.io/creator: admin
    pv.kubernetes.io/bind-completed: 'yes'
    pv.kubernetes.io/bound-by-controller: 'yes'
    volume.beta.kubernetes.io/storage-provisioner: cluster.local/nfs-client-nfs-client-provisioner
  finalizers:
   - kubernetes.io/pvc-protection
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  volumeName: sc-pv-sinfcloud-data
  storageClassName: nfs-client
  volumeMode: Filesystem
```

### K8sYaml

```
kind: Deployment
apiVersion: apps/v1
metadata:
  name: remp-nrp-tpi
  namespace: remp
spec:
  replicas: 1
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 25%
    type: RollingUpdate
  selector:
    matchLabels:
      app: remp-nrp-tpi
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: remp-nrp-tpi
    spec:
      imagePullSecrets:
        - name: remp-harbor-secret
      containers:
        - name: consumer-public-ability-nrp-tpi-app
          image: hub.sinfcloud.com/remp/consumer-public-ability-nrp-tpi-v1:latest
          imagePullPolicy: Always
          env:
            - name: TZ
              value: Asia/Shanghai
            - name: NACOS_URL
              value: ibase-nacos-headless.ibase-kingbase:8848
            - name: REGISTRY_MODE
              value: nacos
            - name: CONFIG_USERNAME
              value: ibaseconf
            - name: CONFIG_PASSWORD
              value: super56&*9
          resources: {}
          command: ["/bin/sh"]
          args: ["-c","java -Djava.security.egd=file:/dev/./urandom -jar ./app.jar"]
        - name: consumer-sso-nrp-tpi-app
          image: hub.sinfcloud.com/remp/consumer-sso-nrp-tpi-v1:latest
          imagePullPolicy: Always
          env:
            - name: TZ
              value: Asia/Shanghai
            - name: NACOS_URL
              value: ibase-nacos-headless.ibase-kingbase:8848
            - name: REGISTRY_MODE
              value: nacos
            - name: CONFIG_USERNAME
              value: ibaseconf
            - name: CONFIG_PASSWORD
              value: super56&*9
          resources: {}
          command: ["/bin/sh"]
          args: ["-c","java -Djava.security.egd=file:/dev/./urandom -jar ./app.jar"]
        - name: consumer-user-nrp-tpi-app
          image: hub.sinfcloud.com/remp/consumer-user-nrp-tpi-v1:latest
          imagePullPolicy: Always
          env:
            - name: TZ
              value: Asia/Shanghai
            - name: NACOS_URL
              value: ibase-nacos-headless.ibase-kingbase:8848
            - name: REGISTRY_MODE
              value: nacos
            - name: CONFIG_USERNAME
              value: ibaseconf
            - name: CONFIG_PASSWORD
              value: super56&*9
          resources: {}
          command: ["/bin/sh"]
          args: ["-c","java -Djava.security.egd=file:/dev/./urandom -jar ./app.jar"]
        - name: provider-public-ability-nrp-tpi-app
          image: hub.sinfcloud.com/remp/provider-public-ability-nrp-tpi-v1:latest
          imagePullPolicy: Always
          env:
            - name: TZ
              value: Asia/Shanghai
            - name: NACOS_URL
              value: ibase-nacos-headless.ibase-kingbase:8848
            - name: REGISTRY_MODE
              value: nacos
            - name: CONFIG_USERNAME
              value: ibaseconf
            - name: CONFIG_PASSWORD
              value: super56&*9
          resources: {}
          command: ["/bin/sh"]
          args: ["-c","java -Djava.security.egd=file:/dev/./urandom -jar ./app.jar"]
      restartPolicy: Always
      terminationGracePeriodSeconds: 30

```



### DockerFile

```
#Docker
FROM hub.sinfcloud.com/support/openjdk8-openj9:alpine-slim-jre
#作者邮箱必写
MAINTAINER liulining@supermap.com
#同步时间
RUN ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
#创建一个文件夹
RUN mkdir -p /ThirdParty-PublicAbility-App-v1/agent
#指定一个工作空间
WORKDIR /ThirdParty-PublicAbility-App-v1
#指定端口
EXPOSE 7806
#将其编译之后的jar文件添加
ADD ./target/ThirdParty-PublicAbility-App.jar ./app.jar
#wait-for
ADD ./wait-for ./
RUN chmod +x wait-for
# 添加wait-for-it.sh
ADD ./wait-for-it.sh ./
RUN chmod +x wait-for-it.sh
#运行命令
#CMD java -Djava.security.egd=file:/dev/./urandom -jar app.jar ${JAVA_OPTS: }
ENTRYPOINT ["java","-Djava.security.egd=file:/dev/./urandom","-jar","app.jar"]

```

Arm-aarch64

### DockerCompose

```
version: '2'
services:
  ams-consumer-app:
    image: ${HUB_URL}/${PROJECT_NAME}/ams-consumer-app:${TAG}
    build:
      dockerfile: ./Dockerfile
      context: ./ams-consumer-app

  #  ams-dajy-provider-app:
  #    image: ${HUB_URL}/${PROJECT_NAME}/ams-dajy-provider-app:${TAG}
  #    build:
  #      dockerfile: ./Dockerfile
  #      context: ./ams-dajy-provider-app

  ams-st-provider-app:
    image: ${HUB_URL}/${PROJECT_NAME}/ams-st-provider-app:${TAG}
    build:
      dockerfile: ./Dockerfile
      context: ./ams-st-provider-app

  remp-dj-db-provider-app:
    image: ${HUB_URL}/${PROJECT_NAME}/remp-dj-db-provider-app:${TAG}
    build:
      dockerfile: ./Dockerfile
      context: ./remp-dj-db-provider-app

  #  remp-dj-gis-provider-app:
  #    image: ${HUB_URL}/${PROJECT_NAME}/remp-dj-gis-provider-app:${TAG}
  #    build:
  #      dockerfile: ./Dockerfile
  #      context: ./remp-dj-gis-provider-app
  #
  remp-dj-sjsh-consumer-app:
    image: ${HUB_URL}/${PROJECT_NAME}/remp-dj-sjsh-consumer-app:${TAG}
    build:
      dockerfile: ./Dockerfile
      context: ./remp-dj-sjsh-consumer-app

  remp-dj-sjsh-provider-app:
    image: ${HUB_URL}/${PROJECT_NAME}/remp-dj-sjsh-provider-app:${TAG}
    build:
      dockerfile: ./Dockerfile
      context: ./remp-dj-sjsh-provider-app

  remp-dj-sl-consumer-app:
    image: ${HUB_URL}/${PROJECT_NAME}/remp-dj-sl-consumer-app:${TAG}
    build:
      dockerfile: ./Dockerfile
      context: ./remp-dj-sl-consumer-app

  remp-dj-sl-provider-app:
    image: ${HUB_URL}/${PROJECT_NAME}/remp-dj-sl-provider-app:${TAG}
    build:
      dockerfile: ./Dockerfile
      context: ./remp-dj-sl-provider-app

  remp-dj-ywcx-consumer-app:
    image: ${HUB_URL}/${PROJECT_NAME}/remp-dj-ywcx-consumer-app:${TAG}
    build:
      dockerfile: ./Dockerfile
      context: ./remp-dj-ywcx-consumer-app

  remp-dj-ywcx-provider-app:
    image: ${HUB_URL}/${PROJECT_NAME}/remp-dj-ywcx-provider-app:${TAG}
    build:
      dockerfile: ./Dockerfile
      context: ./remp-dj-ywcx-provider-app

  remp-dj-ywgl-consumer-app:
    image: ${HUB_URL}/${PROJECT_NAME}/remp-dj-ywgl-consumer-app:${TAG}
    build:
      dockerfile: ./Dockerfile
      context: ./remp-dj-ywgl-consumer-app

  remp-dj-ywgl-provider-app:
    image: ${HUB_URL}/${PROJECT_NAME}/remp-dj-ywgl-provider-app:${TAG}
    build:
      dockerfile: ./Dockerfile
      context: ./remp-dj-ywgl-provider-app

  #  remp-dj-ywjggl-consumer-app:
  #    image: ${HUB_URL}/${PROJECT_NAME}/remp-dj-ywjggl-consumer-app:${TAG}
  #    build:
  #      dockerfile: ./Dockerfile
  #      context: ./remp-dj-ywjggl-consumer-app
  #
  #  remp-dj-ywjggl-provider-app:
  #    image: ${HUB_URL}/${PROJECT_NAME}/remp-dj-ywjggl-provider-app:${TAG}
  #    build:
  #      dockerfile: ./Dockerfile
  #      context: ./remp-dj-ywjggl-provider-app

  #  smpm-consumer-app:
  #    image: ${HUB_URL}/${PROJECT_NAME}/smpm-consumer-app:${TAG}
  #    build:
  #      dockerfile: ./Dockerfile
  #      context: ./smpm-consumer-app
  #
  #  smpm-dj-arcgis-provider-app:
  #    image: ${HUB_URL}/${PROJECT_NAME}/smpm-dj-arcgis-provider-app:${TAG}
  #    build:
  #      dockerfile: ./Dockerfile
  #      context: ./smpm-dj-arcgis-provider-app
  #
  #  smpm-dj-cg-provider-app:
  #    image: ${HUB_URL}/${PROJECT_NAME}/smpm-dj-cg-provider-app:${TAG}
  #    build:
  #      dockerfile: ./Dockerfile
  #      context: ./smpm-dj-cg-provider-app
  #
  #  smpm-dj-gis-provider-app:
  #    image: ${HUB_URL}/${PROJECT_NAME}/smpm-dj-gis-provider-app:${TAG}
  #    build:
  #      dockerfile: ./Dockerfile
  #      context: ./smpm-dj-gis-provider-app
  #
  #  smpm-dj-jg-provider-app:
  #    image: ${HUB_URL}/${PROJECT_NAME}/smpm-dj-jg-provider-app:${TAG}
  #    build:
  #      dockerfile: ./Dockerfile
  #      context: ./smpm-dj-jg-provider-app
  #
  #  smpm-dj-sl-provider-app:
  #    image: ${HUB_URL}/${PROJECT_NAME}/smpm-dj-sl-provider-app:${TAG}
  #    build:
  #      dockerfile: ./Dockerfile
  #      context: ./smpm-dj-sl-provider-app

  # --------------------------------------------------------------------------------------------------------
  # nrp-tpi 构建 START
  # updateBy lln
  # updateTime 23-08-31
  # 三方接入-公共能力消费者
  ThirdParty-PublicAbility-Consumer-App:
    image: hub.sinfcloud.com/remp/consumer-public-ability-nrp-tpi-v1:latest
    build:
      context: ./ThirdParty-PublicAbility-Consumer-App
      dockerfile: ./Dockerfile
  # 三方接入-单点登录消费者
  ThirdParty-SSO-Consumer-App:
    image: hub.sinfcloud.com/remp/consumer-sso-nrp-tpi-v1:latest
    build:
      context: ./ThirdParty-SSO-Consumer-App
      dockerfile: ./Dockerfile
  # 三方接入-用户体系消费者
  ThirdParty-User-Consumer-App:
    image: hub.sinfcloud.com/remp/consumer-user-nrp-tpi-v1:latest
    build:
      context: ./ThirdParty-User-Consumer-App
      dockerfile: ./Dockerfile
  # 三方接入-公共能力提供者
  ThirdParty-PublicAbility-App:
    image: hub.sinfcloud.com/remp/provider-public-ability-nrp-tpi-v1:latest
    build:
      context: ./ThirdParty-PublicAbility-App
      dockerfile: ./Dockerfile
# nrp-tpi 构建 END
# --------------------------------------------------------------------------------------------------------

```



```
#Docker
FROM 19.15.212.28:31104/tsf_100010035/support/openjdk8-openj9:open-8-jre-arm64
#作者邮箱必写
MAINTAINER liulining@supermap.com
#同步时间
RUN echo "Asia/shanghai" > /etc/timezone
RUN cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
#创建一个文件夹
RUN mkdir -p /ThirdParty-PublicAbility-App-v1/agent
#指定一个工作空间
WORKDIR /ThirdParty-PublicAbility-App-v1
#指定端口
EXPOSE 7806
#将其编译之后的jar文件添加
ADD ./target/ThirdParty-PublicAbility-App.jar ./app.jar
#wait-for-it.sh
ADD ./wait-for-it.sh ./
RUN chmod +x wait-for-it.sh
#运行命令
ENTRYPOINT exec java  -Djava.security.egd=file:/dev/./urandom -jar ./app.jar

```



### jenkinsFile

```
#!/usr/bin/env groovy Jenkinsfile
pipeline {
    agent any
    environment{
        MAVEN_HOME = "/home/maven/apache-maven-3.8.8"
    }
    stages {
        stage('拉取代码') {
            steps {
                checkout([$class: 'SubversionSCM', additionalCredentials: [], excludedCommitMessages: '', excludedRegions: '', excludedRevprop: '', excludedUsers: '', filterChangelog: false, ignoreDirPropChanges: false, includedRegions: '', locations: [[cancelProcessOnExternalsFail: true, credentialsId: 'remp-svn', depthOption: 'infinity', ignoreExternalsOption: true, local: '.', remote: 'http://115.236.9.62:20020/NRP-REMPVE6/trunk/nrp-parent/nrp-tpi']], quietOperation: true, workspaceUpdater: [$class: 'UpdateUpdater']])
            }
        }
        stage('编译代码') {
            steps {
                echo "开始编译打包"
                sh "mvn clean package -Dmaven.test.skip=true -U -T 4"
                echo "编译打包完成"
                sh "rm -rf ./.svn"
            }
        }
        stage('制作镜像') {
            parallel {
                stage('制作消费者【publicAbility-consumer】') {
                    steps {
                        sh "docker build -t hub.sinfcloud.com/remp/consumer-public-ability-nrp-tpi-v1:latest .././nrp-publish/ThirdParty-PublicAbility-Consumer-App"
                    }
                }
                stage('制作消费者【sso-consumer】') {
                    steps {
                        sh "docker build -t hub.sinfcloud.com/remp/consumer-sso-nrp-tpi-v1:latest .././nrp-publish/ThirdParty-SSO-Consumer-App"
                    }
                }
                stage('制作消费者【user-consumer】') {
                    steps {
                        sh "docker build -t hub.sinfcloud.com/remp/consumer-user-nrp-tpi-v1:latest .././nrp-publish/ThirdParty-User-Consumer-App"
                    }
                }
                stage('制作提供者【ThirdParty-Provider-App】') {
                    steps {
                        sh "docker build -t hub.sinfcloud.com/remp/provider-public-ability-nrp-tpi-v1:latest .././nrp-publish/ThirdParty-PublicAbility-App"
                    }
                }
            }
        }
        stage('推送仓库') {
            parallel {
                stage('推送消费者【publicAbility-consumer】'){
                    steps {
                        sh "docker push hub.sinfcloud.com/remp/consumer-public-ability-nrp-tpi-v1:latest"
                    }
                }
                stage('推送消费者【sso-consumer】'){
                    steps {
                        sh "docker push hub.sinfcloud.com/remp/consumer-sso-nrp-tpi-v1:latest"
                    }
                }
                stage('推送消费者【user-consumer】'){
                    steps {
                        sh "docker push hub.sinfcloud.com/remp/consumer-user-nrp-tpi-v1:latest"
                    }
                }
                stage('推送提供者【publicAbility-provider】'){
                    steps {
                        sh "docker push hub.sinfcloud.com/remp/provider-public-ability-nrp-tpi-v1:latest"
                    }
                }
            }
        }
        stage('更新服务'){
            steps {
                sshPublisher(publishers: [sshPublisherDesc(configName: 'kubernetes-master', transfers: [sshTransfer(cleanRemote: false, excludes: '',
                        execCommand: 'kubectl rollout restart deployment remp-nrp-tpi -n remp && kubectl rollout status deployment remp-nrp-tpi -n remp',
                        execTimeout: 120000, flatten: false, makeEmptyDirs: false, noDefaultExcludes: false, patternSeparator: '[, ]+', remoteDirectory: '', remoteDirectorySDF: false, removePrefix: '', sourceFiles: '')], usePromotionTimestamp: false, useWorkspaceInPromotion: false, verbose: true)])
            }
        }
    }
}

```



# 四、腾讯云使用

TKE

https://cloud.tencent.com/document/product/457

TSF

https://cloud.tencent.com/document/product/649

# 五、资源申请
基于在研发环境的压测情况，和裸数据库的压测情况，结合各个市租户的历史的并发数量，进行估算。
提供申请资源量的支撑材料，
按一个租户（市县）用户200人，因登记中心业务时间较为集中，故按50%人数计算并发量（200*50%=100），
一个租户互联网用户2000，按5%计算并发量（2000*5%=100），
故一个市的并发量为200，一期上线3个市共600并发，按照相应公式申请如下容器资源：

| 业务名称           | 容器名             | 负载 | 配置 | 利用率 | 容器数量 |
| ------------------ | ------------------ | ---- | ---- | ------ | -------- |
| 全局管理服务       | 全局管理服务       | CPU  | 16核 | 37.89% | 1        |
|                    |                    | 内存 | 32 G | 23.28% |          |
| 访问控制服务       | 访问控制服务       | CPU  | 16核 | 59.21% | 1        |
|                    |                    | 内存 | 32 G | 25.24% |          |
| 工作流服务         | 工作流服务         | CPU  | 16核 | 56.68% | 1        |
|                    |                    | 内存 | 32 G | 19.2%  |          |
| 编码中心服务       | 编码中心服务       | CPU  | 16核 | 28.74% | 1        |
|                    |                    | 内存 | 32 G | 21.03% |          |
| 单点登录服务       | 单点登录服务       | CPU  | 16核 | 76.15% | 1        |
|                    |                    | 内存 | 32 G | 18.73% |          |
| 不动产登记系统服务 | 不动产登记系统服务 | CPU  | 16核 | 23.24% | 1        |
|                    |                    | 内存 | 32 G | 26.99% |          |
| 地籍系统服务       | 地籍系统服务       | CPU  | 16核 | 34.28% | 1        |
|                    |                    | 内存 | 32 G | 15.79% |          |
| 通用组件服务       | 通用组件服务       | CPU  | 16核 | 35.46% | 1        |
|                    |                    | 内存 | 32 G | 28.81% |          |
| 档案系统服务       | 档案系统服务       | CPU  | 16核 | 21.26% | 1        |
|                    |                    | 内存 | 32 G | 19.51% |          |



| 中文名称           | CPU    | 内存   | 测试规格 | 测试场景 | 申请规格 | 申请数量 | 申请说明                                                     |
| ------------------ | ------ | ------ | -------- | -------- | -------- | -------- | ------------------------------------------------------------ |
| 全局管理服务       | 37.89% | 23.28% | 16C 32GB | 100      | 8C 16GB  | 5        | 以100并发进行测试，CPU使用37.89%，内存使用23.28%，按比例得出600并发时，该场景所需资源为：37.89%* 6=227.34%（CPU），转换为CPU核数约为16C*227.34%=36C，23.28%*6=139.68%（内存），转换为内存约为32GB*139.68%=45GB，故该场景需要申请5个 8核CPU、内存16 G 容器较为符合场景需求。 |
| 访问控制服务       | 59.21% | 25.24% | 16C 32GB | 100      | 16C 16GB | 4        | 以100并发进行测试，CPU使用59.21%，内存使用25.24%，按比例得出600并发时，该场景所需资源为：59.21%* 6=355.26%（CPU），转换为CPU核数约为16C*355.26%=57C，25.24%*6=151.44%（内存），转换为内存约为32GB*151.44%=48GB，该服务CPU资源使用率较高，故该场景需要申请4个 16核CPU、内存16 G 容器较为符合场景需求。 |
| 工作流服务         | 56.68% | 19.2%  | 16C 32GB | 100      | 16C 16GB | 4        | 以100并发进行测试，CPU使用56.68%，内存使用19.2%，按比例得出600并发时，该场景所需资源为：56.68%* 6=340.08%（CPU），转换为CPU核数约为16C*340.08%=54C，19.2%*6=115.2%（内存），转换为内存约为32GB*115.2%=37GB，该服务CPU资源使用率较高，故该场景需要申请4个 16核CPU、内存16 G 容器较为符合场景需求。 |
| 编码中心服务       | 28.74% | 21.03% | 16C 32GB | 100      | 8C 16GB  | 4        | 以100并发进行测试，CPU使用28.74%，内存使用21.03%，按比例得出600并发时，该场景所需资源为：28.74%* 6=172.44%（CPU），转换为CPU核数约为16C*172.44%=28C，21.03%*6=126.18%（内存），转换为内存约为32GB*126.18%=40GB，该服务CPU资源使用率较高，故该场景需要申请4个 8核CPU、内存16 G 容器较为符合场景需求。 |
| 单点登录服务       | 76.15% | 18.73% | 16C 32GB | 100      | 32C 16GB | 3        | 以100并发进行测试，CPU使用28.74%，内存使用21.03%，按比例得出600并发时，该场景所需资源为：76.15%* 6=456.9%（CPU），转换为CPU核数约为16C*456.9%=73C，18.73%*6=112.38%（内存），转换为内存约为32GB*112.38%=36GB，该服务CPU资源使用率较高，故该场景需要申请4个 32核CPU、内存16 G 容器较为符合场景需求。 |
| 不动产登记系统服务 | 23.24% | 26.99% | 16C 32GB | 100      | 8C 16GB  | 4        | 以100并发进行测试，CPU使用23.24%，内存使用26.99%，按比例得出600并发时，该场景所需资源为：23.24%* 6=139.44%（CPU），转换为CPU核数约为16C*139.44%=22C，26.99%*6=161.94%（内存），转换为内存约为32GB*161.94%=52GB，该服务CPU资源使用率较高，故该场景需要申请4个 8核CPU、内存16 G 容器较为符合场景需求。 |
| 地籍系统服务       | 34.28% | 15.79% | 16C 32GB | 100      | 16C 16GB | 3        | 以100并发进行测试，CPU使用34.28%，内存使用15.79%，按比例得出600并发时，该场景所需资源为：34.28%* 6=205.68%（CPU），转换为CPU核数约为16C*205.68%=33C，15.79%*6=94.74%（内存），转换为内存约为32GB*94.74%=30GB，该服务CPU资源使用率较高，故该场景需要申请4个 16核CPU、内存16 G 容器较为符合场景需求。 |
| 通用组件服务       | 35.46% | 28.81% | 16C 32GB | 100      | 16C 32GB | 3        | 以100并发进行测试，CPU使用35.46%，内存使用28.81%，按比例得出600并发时，该场景所需资源为：35.46%* 6=212.76%（CPU），转换为CPU核数约为16C*212.76%=34C，28.81%*6=172.86%（内存），转换为内存约为32GB*172.86%=55GB，该服务CPU资源使用率较高，故该场景需要申请4个 16核CPU、内存32 G 容器较为符合场景需求。 |
| 档案系统服务       | 21.26% | 19.51% | 16C 32GB | 100      | 8C 16GB  | 3        | 以100并发进行测试，CPU使用21.26%，内存使用19.51%，按比例得出600并发时，该场景所需资源为：21.26%* 6=127.56%（CPU），转换为CPU核数约为16C*127.56%=20C，19.51%*6=117.06%（内存），转换为内存约为32GB*117.06%=37GB，该服务CPU资源使用率较高，故该场景需要申请3个 8核CPU、内存16 G 容器较为符合场景需求。 |

# 六、测试和监控

测试

各个微服务是否正常启动

业务中台能否正常使用，各个流程都走一遍看有无报错

网络通信正常与否

监控

研发环境

Prometheus+Grafana

https://zhuanlan.zhihu.com/p/321286313

https://www.jianshu.com/p/8d2c020313f0

云环境

云平台自带监控
