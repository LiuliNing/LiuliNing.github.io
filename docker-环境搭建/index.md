# Dokcer研发环境搭建记录


<!--more-->


# 一 Ubuntu1804-Dokcer

```
Author: JDZT.lln
CreateDate: 2020-12-09
UpdateDate: 2021-10-17
说明:Ubuntu版本:18.04.5 LTS
```

## 1.1 安装及配置

### 1.1.1 安装docker和docker-compose

```
#安装docker 
sudo curl -fsSL https://get.docker.com | bash -s docker --mirror Aliyun 

#安装docker-compose 
sudo curl -L https://get.daocloud.io/docker/compose/releases/download/1.22.0/docker-compose- `uname -s`-`uname -m` -o /usr/local/bin/docker-compose 
sudo curl -L https://github.com/docker/compose/releases/download/v2.18.1/docker-compose-linux-x86_64-$(uname -s)-$(uname -m) -o ./docker-compose-linux-x86_64



#授权docker-compose 
sudo chmod +x /usr/local/bin/docker-compose
```

### 1.1.2：配置docker镜像加速

编辑docker配置文件，若没有该文件则新建

```
参考文档
https://help.aliyun.com/document_detail/60750.html?spm=a2c4g.11186623.6.545.OY7haW
```

新建文件

```
cd /etc/docker
touch daemon.json
```

编辑文件

```
vim /etc/docker/daemon.json
```

加入配置信息，不同的云端(阿里|腾讯)配置不同的镜像，可配置多个镜像

```
{ 
	"registry-mirrors": ["https://docker.mirrors.ustc.edu.cn","https://mirror.ccs.tencentyun.com"] 
}
{ 
	"log-driver":"json-file", "log-opts": {"max-size":"500m", "max-file":"3"} 
}
```

使配置生效

```
systemctl daemon-reload
systemctl restart docker
```

### 1.1.3：打包项目并在Docker中运行

流程：打包项目jar包，编写Dockerfile文件，放入到同一个文件夹中，使用docker命令打包，运行，就可以外部访问

Dockerfile 编写说明

```
# 基础镜像使用java
FROM java:8
# 作者
MAINTAINER jd-lln
# 定义卷
VOLUME /tmp
# 将jar包添加到容器中并更名为test1.jar
ADD easysite-exploration.jar easysite-exploration.jar 
# 指定容器内要暴露的端口
EXPOSE 8013
RUN echo "Asia/Shanghai" > /etc/timezone
ENTRYPOINT [ "sh", "-c", "java -jar easysite-exploration.jar" ]

```

```
docker打包命令
在放置jar包和Dockerfile的目录下，使用命令打包为镜像（注意命令后面有个 '.',xxx/xxx为镜像名称，v1为版本号）
docker build -t xxx/xxx:v1 . 
```



1.2：在Docker中安装ubuntu[~v.v~]

拉取 Ubuntu 镜像

```
拉取最新镜像
docker pull ubuntu:latest
拉取指定镜像
docker pull ubuntu:18.04
```

在docker的ubuntu中安装jdk

```
复制宿主机中的/java/jdk8.tar.gz 文件至 docker中ubuntu中的/java 文件夹中，ea49f55dde3d为镜像id
docker cp /java/jdk8.tar.gz ea49f55dde3d:/java/

删除原装的openjdk
sudo apt-get purge openjdk*

解压文件、配置环境变量略
```

！！！特别注意！！！

一定要提交对原镜像的修改，打包成新的镜像（否则在重启原镜像后，无法保留在原镜像上的所有修改，包含各种配置和安装的资源）

```
docker commit -a="xxx" -m="test" c8314e7588ad xxx/test:0.1
打包命令          作者    描述信息  原镜像id   新的名字 版本号 
```





# 二 Centos7-Docker

## 1 安装docker

```
参考
https://blog.csdn.net/qq_26400011/article/details/113856681

1、安装之前现卸载系统上原有的Docker
yum remove docker \
docker-client \
docker-client-latest \
docker-common \
docker-latest \
docker-latest-logrotate \
docker-logrotate \
docker-engine

2、安装需要的安装包yum-utils
yum install -y yum-utils
3、设置镜像仓库地址
阿里云的镜像仓库地址
yum-config-manager \
--add-repo \
http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
4、安装docker相关的引擎
yum makecache fase 


```

## 2 设置镜像仓库

```
vi /etc/docker/daemon.json 
添加 镜像路径
xxx为镜像url

{
  "registry-mirrors": ["xxx"]
}

{
  "registry-mirrors": ["https://nmg8n4ak.mirror.aliyuncs.com"]
}

# 中国科技大学       
https://docker.mirrors.ustc.edu.cn
# 网易 
http://hub-mirror.c.163.com
# 阿里云（需要注册账户来进行个人镜像路径获取）


```

# 三 docker软件安装

## 3 安装mysql

```
安装最新版本
docker pull mysql:8.0.27
启动容器，将3306端口映射至外部3306端口，设置进入密码123456
docker run \
--privileged=true \
--restart=always \
--name mysql:8.0.27 -p 3306:3306 \
-e MYSQL_ROOT_PASSWORD=123456 \
-v /opt/mysql/data:/var/lib/mysql \
-v /opt/mysql/conf.d:/etc/mysql/conf.d \
-d mysql \
--lower_case_table_names=1 

参数说明



进入容器，配置mysql   
***为容器id
docker exec -it mysql /bin/bash

进入mysql，修改远程连接
mysql -uroot -p123456

更新root权限
update user set host='%' where user='root';
alter user 'root'@'%' identified by '123456' password expire never;
alter user 'root'@'%' identified with mysql_native_password by '123456';

刷新权限配置
flush privileges;

重启mysql后使用navicat工具测试连接
```

## 4 安装redis

```
# 安装最新redis
docker pull redis
# 启动redis镜像
docker run -p 6379:6379 \
--restart=always \
--name redis \
-v /opt/redis/conf/redis.conf:/etc/redis/redis.conf \
-v /opt/redis/data:/data \
-d redis redis-server /etc/redis/redis.conf \
--appendonly yes





# 启动参数说明
-p 6379:6379 端口映射：前表示主机部分，：后表示容器部分。
--name redis  指定该容器名称，查看和进行操作都比较方便。
-v 挂载目录，规则与端口映射相同。
-d redis 表示后台启动redis
redis-server /etc/redis/redis.conf  以配置文件启动redis，加载容器内的conf文件（其实找到的是挂载的宿主机映射目录/opt/redis/redis.conf）
appendonly yes 开启redis 持久化

# 设置自启动
docker update --restart=always 镜像启动id
```

## 5 安装nginx

```
# 下载最新nginx
docker pull nginx

# 将本地主配置文件、从配置文件、静态资源文件映射到容器内部，并启动容器
docker run --name nginx \
-p 80:80 \
-p 6001:6001 \
-p 7001:7001 \
-p 7070:7070 \
-p 10011:10011 \
--restart=always \
--privileged=true \
-v /opt/nginx/docker.nginx.conf:/etc/nginx/nginx.conf \
-v /opt/nginx/docker-nginx-confs/:/opt/nginx/docker-nginx-confs/  \
-v /www/wwwroot/dists/:/www/wwwroot/dists \
-d nginx

 docker run -p 80:80 -p 6001:6001 -p6002:6002 -p 7001:7001 -p 7070:7070 -p 10011:10011 --name nginx --restart=always --privileged=true -v /opt/nginx/docker.nginx.conf:/etc/nginx/nginx.conf -v /opt/nginx/docker-nginx-confs/:/opt/nginx/docker-nginx-confs/  -v /www/wwwroot/dists/:/www/wwwroot/dists -d nginx

```

## 6安装可视化界面 portainer

```
#下载镜像
dk pull portainer/portainer
#启动镜像
docker run -d -p 9000:9000 \
--restart=always \
-v /var/run/docker.sock:/var/run/docker.sock \
--name prtainer \
docker.io/portainer/portainer
```

## 7安装elasticsearch

```
#设置max_map_count的值
sysctl -w vm.max_map_count=262144

#拉取镜像
docker pull elasticsearch:7.7.0

#启动镜像
docker run --name els -d -e \
ES_JAVA_OPTS="-Xms512m -Xmx512m" -e "discovery.type=single-node" \
-p 9200:9200 -p 9300:9300 elasticsearch:7.7.0

#访问
http://ip:9200/
出现以下内容代表启动成功
{
  "name" : "80e63ddf710f",
  "cluster_name" : "docker-cluster",
  "cluster_uuid" : "lEabtVF1Tfqgny2MCud7aA",
  "version" : {
    "number" : "7.7.0",
    "build_flavor" : "default",
    "build_type" : "docker",
    "build_hash" : "81a1e9eda8e6183f5237786246f6dced26a10eaf",
    "build_date" : "2020-05-12T02:01:37.602180Z",
    "build_snapshot" : false,
    "lucene_version" : "8.5.1",
    "minimum_wire_compatibility_version" : "6.8.0",
    "minimum_index_compatibility_version" : "6.0.0-beta1"
  },
  "tagline" : "You Know, for Search"
}

#安装elasticsearch-head 客户端可视化工具
docker pull mobz/elasticsearch-head:5-alpine
docker run -d \
  --name=elasticsearch-head \
  --restart=always \
  -p 9100:9100 \
  docker.io/mobz/elasticsearch-head:5-alpine
```

## 8搭建nexus私服

```
#参考博客
https://blog.csdn.net/u012943767/article/details/79475718

```

```
#拉取最新镜像
docker pull sonatype/nexus3

#运行镜像 挂载磁盘信息、端口8081
docker run -d -p 8081:8081 \
--name nexus \
--restart=always \
-v /opt/nexus/data:/var/nexus-data \
sonatype/nexus3

docker run -d --name nexus3 \
--privileged=true \
--restart=always \
-p 8081:8081 \
-p 10000:10000 \
-p 10010:10010 \
-p 10020:10020 \ 
-v /opt/nexus3/nexus-data:/var/nexus-data \
fe48ec3e44c6

dk exec -it nexus3 /bin/bash
dk cp nexus3:/nexus-data/  /opt/nexus/nexus-data/
#登录
默认账号：admin
默认密码：
	默认密码在容器内部的 /nexus-data/admin.password  文件中
	进入容器获取复制密码
#建立仓库
#设置中文

8081端口是nexus3的服务端口
10000端口用于host镜像仓库的服务端口
10010端口用于group镜像仓库的服务端口




```

## 9搭建jenkins自动化部署工具

```
参考博客
https://blog.csdn.net/cristianoxm/article/details/125740894?ops_request_misc=&request_id=&biz_id=102&utm_term=java.net.UnknownHostException:&utm_medium=distribute.pc_search_result.none-task-blog-2~all~sobaiduweb~default-5-125740894.nonecase&spm=1018.2226.3001.4187
```



```
#下载镜像
docker pull jenkins/jenkins
#搭建目录
mkdir -p /data/jenkins
chmod 777 /data/jenkins
#启动镜像
docker run -d \
-p 10240:8080 \
-p 10241:50000 \
-v /data/jenkins:/var/jenkins_home \
-v /etc/localtime:/etc/localtime \
--name jenkins jenkins/jenkins
#获取密码
docker logs jenkins
#修改镜像
vim /data/jenkins/hudson.model.UpdateCenter.xml
#替换url
https://mirrors.tuna.tsinghua.edu.cn/jenkins/updates/update-center.json
#启动jenkins后，生成updates目录，进入目录
cd /data/jenkins/updates
#替换源
sed -i 's/https:\/\/updates.jenkins.io\/download/http:\/\/mirrors.tuna.tsinghua.edu.cn\/jenkins/g' /data/jenkins/updates/default.json && sed -i 's/http:\/\/www.google.com/https:\/\/www.baidu.com/g' /data/jenkins/updates/default.json
#正式使用




```

## 10搭建sonarqube代码检测工具

```
#拉取postgres数据库
docker pull postgres 

#拉取sonarqube
docker pull sonarqube

#启动数据库
docker run \
--restart=always \
--name postgres \
-v /opt/postgres/data:/var/lib/postgresql/data \
-e POSTGRES_PASSWORD=root \
-p 5432:5432 \
-d postgres

#启动sonarqube
docker run \
--name sonar \
--restart=always \
--link postgres \
--privileged=true \
--restart=always \
-e SONARQUBE_JDBC_URL=jdbc:postgresql://postgres:5432/sonar \
-e SONARQUBE_JDBC_USERNAME=postgres \
-e SONARQUBE_JDBC_PASSWORD=root \
-v /opt/sonarqube/data:/var/lib/postgresql/data \
-v /opt/sonarqube/extensions:/opt/sonarqube/extensions \
-v /opt/sonarqube/logs:/opt/sonarqube/logs  \
-v /opt/sonarqube/conf:/opt/sonarqube/conf \
-p 9003:9000 \
-d sonarqube



注意事项和异常处理：
1.通过 -e 填入数据库账号密码
2.需要在postgres中提前创建数据库sonar
3.url <postgres:5432>中的postgres为link的数据库镜像名称，不能为localhost或127.0.0.1
4.需要设置java环境
5.启动报错，logs日志中关键词：
max virtual memory areas vm.max_map_count [65530] is too low, increase to at least [262144]
解决办法：
切换到root用户修改配置sysctl.conf
vi /etc/sysctl.conf 
添加下面配置
vm.max_map_count=655360
并执行命令
sysctl -p
汉化插件安装失败
从gitee上下载符合版本的汉化包（sonar-l10n-zh-plugin-10.0.jar），放入plugins目录中
检测报告pdf导出
从gitee上下载pdf插件（sonar-pdfreport-plugin-4.0.1.jar），放入plugins目录中

#使用代码检测
mvn compile sonar:sonar \
  -Dsonar.projectKey=plm \
  -Dsonar.host.url=http://10.10.10.8:9001 \
  -Dsonar.login=c531b16a1d0dbb369fa7dfb79890902264017ef6

```



## 11使用IDEA连接ubuntu中的docker容器

```
#参考链接
https://blog.csdn.net/qq_43107323/article/details/102854705

#编辑配置文件
sudo vim /lib/systemd/system/docker.service

#添加开发2375端口
在[Service]标签中，ExecStart行，末尾sock后面, 添加 
-H tcp://0.0.0.0:2375

#重新加载配置
systemctl daemon-reload

#查看端口配置是否启动
netstat -tulp

#安装idea的docker插件

#进入docker设置，选择tcp连接，输入
tcp://虚拟机ip:2375

#可以再idea的服务中看到已经连接的docker容器内容

```



## 12安装cnnal

```
#拉取最新canal
docker pull canal/canal-server:latest

#运行canal容器（导出配置文件后关闭）
docker run -p 11111:11111 --name canal -d canal/canal-server

#复制容器中文件至指定目录
dk cp canal:/home/admin/canal-server /opt/canal/

#重新带配置映射启动
docker run \
--name canal \
-e canal.instance.master.address=127.0.0.1:3306 \
-e canal.instance.dbUsername=canal \
-e canal.instance.dbPassword=canal \
-v /opt/canal/canal-server:/home/admin/canal-server \
-p 11111:11111 \
-d canal/canal-server
```

## 13安装kkfile

```
#拉取最新
docker pull keking/kkfileview:v4.0.0
#运行
docker run \
--name=kkfile \
--privileged=true \
--restart=always \
-v /opt/minio/data:/opt/minio/data \
-v /tmp/:/tmp/ \
-v /opt/kkfile/kkFileView-3.3.1:/opt/kkFileView-3.3.1 \
-p 8012:8012 \
-d keking/kkfileview:v3.3.1


docker run \
--name=kkfile \
--privileged=true \
--restart=always \
-v /opt/minio/data:/opt/minio/data \
-v /tmp/:/tmp/ \
-v /opt/kkfile/kkFileView-4.0.0:/opt/kkFileView-4.0.0 \
-p 18012:8012 \
-d keking/kkfileview:v4.0.0


dk exec -it kkfile /bin/bash
kkFileView-4.0.0/
libreoffice7.1/


dk cp kkfile:/opt/kkFileView-4.0.0/ /opt/kkfile/
/opt/openoffice4/program/soffice -headless -accept="socket,host=127.0.0.1,port=8100;urp;" -nofirststartwizard &



```



## 14安装Gitea

```
#拉取最新
docker pull gitea/gitea
#运行
docker run -d \
--privileged=true \
--restart=always \
--name=gitea \
-p 10025:22 \
-p 10026:3000 \
-v /opt/gitea:/data \
gitea/gitea:latest

```

## 15安装tomcat

```
docker pull tomcat

docker run -d \
--privileged=true \
--restart=always \
--name=tomcat \
-p 12000:12000 \
-v /opt/tomcat_map/:/usr/local/tomcat/ \
-v /usr/share/fonts/:/usr/share/fonts/ \
tomcat:latest
```

## 16安装showdoc

```
#拉取最新
docker pull registry.cn-shenzhen.aliyuncs.com/star7th/showdoc
#改名tag
docker tag registry.cn-shenzhen.aliyuncs.com/star7th/showdoc:latest lln/showdoc:latest 
#启动showdoc容器
docker run -d --name showdoc \
--user=root \
--privileged=true \
--restart=always \
-p 5000:80 \
-v /opt/showdoc/html:/var/www/html/ lln/showdoc
```

## 17 安装minio

```
docker run -p 10123:9000 --name minio \
  -e "MINIO_ACCESS_KEY=admin" \
  -e "MINIO_SECRET_KEY=12345678" \
  -v /opt/minio/data:/data \
  -v /opt/minio/config:/root/.minio \
  -d \
 minio/minio:RELEASE.2021-06-14T01-29-23Z server /data
```

## 18 安装clickhouse

```

```

## 19 安装harbor



## 20 安装rancher

```
https://docs.rancher.cn/rancher2.5/

sudo docker run --privileged -d --restart=unless-stopped -p 80:80 -p 443:443 rancher/rancher:stable
```

## 21 linux命令搜索

```
https://github.com/jaywcjlove/linux-command

docker pull wcjiang/linux-command
# Or
docker pull ghcr.io/jaywcjlove/linux-command:latest
docker run --name linux-command --rm -d -p 9665:3000 wcjiang/linux-command:latest
# Or
docker run --name linux-command -itd -p 9665:3000 wcjiang/linux-command:latest
# Or
docker run --name linux-command -itd -p 9665:3000 ghcr.io/jaywcjlove/linux-command:latest

http://192.168.11.127:9665/
```

```
docker pull wcjiang/linux-command
# Or
docker pull ghcr.io/jaywcjlove/linux-command:latest
docker run --name  linux-command --rm -d -p 9665:3000 wcjiang/linux-command:latest
# Or
docker run --name linux-command -itd -p 9665:3000 wcjiang/linux-command:latest
# Or
docker run --name linux-command -itd -p 9665:3000 ghcr.io/jaywcjlove/linux-command:latest
```



## 22 安装oracle19c

```
# 下载镜像
docker pull registry.cn-hangzhou.aliyuncs.com/zhuyijun/oracle:19c

mkdir -p /home/oracle/data
chmod 777 /home/oracle/data/

docker run -d  \
-p 1524:1521 -p 5502:5500 \
-e ORACLE_SID=ORCLCDB \
-e ORACLE_PDB=ORCLPDB1 \
-e ORACLE_PWD=supermap1234! \
-e ORACLE_EDITION=standard \
-e ORACLE_CHARACTERSET=AL32UTF8 \
-v /home/oracle/data:/opt/oracle/oradata \
--name orcl19c \
registry.cn-hangzhou.aliyuncs.com/zhuyijun/oracle:19c

```



#   四 Docker常用命令

```
启动docker
sudo service docker start

停止docker
sudo service docker stop

批量停止docker容器
docker ps | grep sinfcloud | awk '{print $1}' | xargs docker stop
说明 批量停止包含 sinfcloud 关键字的镜像，并打印 容器 id

批量删除已停止的docker镜像（docker版本大于1.3使用）
sudo docker container prune

批量删除包含某个关键字的镜像（）
docker rmi $(docker images | grep 'sinfcloud')
说明 批量删除所有包含 sinfcloud 的镜像包


重启docker
sudo service docker restart

列出机器上的容器
docker images

查看所有容器
docker ps -a

启动容器
docker run -itd 容器名称或镜像编号
启动容器并将容器内端口2222映射到外部宿主机端口1111
docker run -itd -p 1111:2222

进入容器
docker attach 容器编号（使用 attach 命令有一个问题。当多个窗口同时使用该命令进入该容器时，所有的窗口都会同步显示。如果有一个窗口阻塞了，那么其他窗口也无法再进行操作。）

关闭容器
docker stop 容器编号

在容器中开启一个交互模式的终端:
xxx 为 镜像id
docker exec -it xxx /bin/bash

删除容器 容器名称:标签
docker rmi REPOSITORY:TAG 
常见错误:
"Error response from daemon: conflict: unable to delete a838c1b6a0e4 (cannot be forced) - image has dependent child images"
或
"Error response from daemon: conflict: unable to remove repository reference "ubuntu-1215:V0.1" (must force) - container 0257eaaf90a4 is using its referenced image a838c1b6a0e4"
使用以下命令后，再次执行删除镜像命令

删除所有停止的容器
docker rm $(docker ps -a -q)

在docker反复build后，会存留很多none镜像，下面命令一键删除所有none镜像
docker rmi `docker images | grep  '<none>' | awk '{print $3}'`

提交修改后的镜像
docker commit -a="xxx" -m="test" c8314e7588ad xxx/test:0.1
打包命令          作者    描述信息  原镜像id   新的名字 版本号 

docker commit -a="jdzt.lln" -m="add:vim" dc6133ec39d8 nginx:0.3

删除多余tag
docker rmi 仓库名称:TAG

设置自启动
docker update --restart=always ID

# 查看本机docker所有网络
docker network ls

# 创建网络  --driver 是网络模式，--subnet是子网掩码，后面的/16表示可以生成6万多个ip， --gateway是网关地址
docker network create 网络名称 --driver bridge --subnet 192.168.0.1/16 --gateway 192.168.0.2 mynet

# 将容器加入到当前网络
docker network connect 网络名称 容器名称

# 断开容器的网络 （容器必须是运行状态才能断开连接）
docker network disconnect 网络名称 容器名称

# 查看网络的详细信息
docker network inspect 网络id/网络名称

#删除网络
docker network rm 网络id/网络名称

# 删除所有未使用的网络
docker network prune --f



```

# 五 Dockerfile

```
# 基础镜像使用java
FROM java:8

# 作者
MAINTAINER jd-lln

# VOLUME：用于指定持久化目录
VOLUME /tmp

# ADD：将本地文件添加到容器中，tar类型文件会自动解压(网络压缩资源不会被解压)，可以访问网络资源，类似wget
ADD test.jar app.jar 

# EXPOSE：指定于外界交互的端口
EXPOSE 9999

# RUN：构建镜像时执行的命令
RUN echo "Asia/Shanghai" > /etc/timezone

# ENTRYPOINT：配置容器，使其可执行化。配合CMD可省去"application"，只使用参数。
ENTRYPOINT ["java","-Djava.security.egd=file:/dev/./urandom","-jar","app.jar"]

#构建镜像
docker build -t plm .
#运行镜像
dk run --name=plm \
-p 6003:6003 \
--privileged=true \
-d plm 

```

# 六 常见问题

```
基础镜像没有常用命令
    配置 镜像
    vim /etc/apt/sources.list
    复制阿里云镜像,替换原国外镜像
    deb http://mirrors.aliyun.com/ubuntu/ trusty main restricted universe multiverse 
    deb http://mirrors.aliyun.com/ubuntu/ trusty-security main restricted universe multiverse 
    deb http://mirrors.aliyun.com/ubuntu/ trusty-updates main restricted universe multiverse 
    deb http://mirrors.aliyun.com/ubuntu/ trusty-proposed main restricted universe multiverse 
    deb http://mirrors.aliyun.com/ubuntu/ trusty-backports main restricted universe multiverse 
    deb-src http://mirrors.aliyun.com/ubuntu/ trusty main restricted universe multiverse 
    deb-src http://mirrors.aliyun.com/ubuntu/ trusty-security main restricted universe multiverse 
    deb-src http://mirrors.aliyun.com/ubuntu/ trusty-updates main restricted universe multiverse 
    deb-src http://mirrors.aliyun.com/ubuntu/ trusty-proposed main restricted universe multiverse 
    deb-src http://mirrors.aliyun.com/ubuntu/ trusty-backports main restricted universe multiverse
```

```
常用命令缺失
    安装vim
        apt-get update
        apt-get install vim
    安装ping ifconfig
        apt-get update
        apt install iputils-ping    # ping
        apt install net-tools       # ifconfig 

```



# 七 出现的异常及解决方案

### 1.docker 运行时No chain/target/match by that name

```
启动镜像出现iptables: No chain/target/match by that name
```

解决方案

```
service docker restart
或
systemctl restart  docker
```

### 2.安装sonar启动镜像报错

```
日志内容
max virtual memory areas vm.max_map_count [65530] is too low, increase to at least [262144]

解决办法：
切换到root用户修改配置sysctl.conf
vi /etc/sysctl.conf 
添加下面配置：
vm.max_map_count=655360
并执行命令：
sysctl -p

```

### 3，容器内部时区错误

```
1.复制宿主机上的zoneinfo文件夹到容器下的/usr/share/
docker cp /usr/share/zoneinfo ef761110f5a2:/usr/share/

2.执行
docker exec -it detectronwj /bin/bash
进入dokcer内后，执行以下操作：
ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
echo "Asia/Shanghai" > /etc/timezone
输入验证
date

2.compose脚本中指定时区（推荐）
```

 
