### 关键词
LXC Docker架构 仓库注册服务器 仓库 镜像 容器 UnionFS bootfs rootfs registry repo image container DockFile tomact
## 为什么轻？
![LXC原理](https://upload-images.jianshu.io/upload_images/5344773-022c56b9a1df1f5e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
1. 去掉了传统VM架构中Hypervisor(硬件资源虚拟化)这一层，Docker Engine 直接对宿主系统进行库调用，不关心具体硬件实现;
2. 去掉庞大虚拟机操作系统(GuestOS)，进程级别进行隔离，极大节省运行空间，扩容响应效率。

## Docker架构
Docker使用C/S架构，Client 通过接口与Server进程通信实现容器的构建，运行和发布。client和server可以运行在同一台集群，也可以通过跨主机实现远程通信。
![Docker架构](https://upload-images.jianshu.io/upload_images/5344773-84bf22d6e6e98f2b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 核心概念
仓库注册服务器 registry
```
为仓库提供托管的服务器，目前最大镜像仓库托管服务器 DockerHub，
还有提供加速服务的诸如阿里云镜像加速服务器、网易云加速服务器 等都属于这一类。
```
镜像仓库 repo
```
仓库注册服务器中镜像仓库往往以命名空间方式存在，可类比为linux服务器下众多文件目录。
镜像仓库下往往存在多个镜像。不同镜像依据 tag 划分为不同版本。
```
镜像 image
```
一堆类似于“花卷”的只读层，以java类模板类比，基于同一份镜像(类)，可以创建不同容器(对象)。
```
 ![只读层](https://upload-images.jianshu.io/upload_images/5344773-656152c8139d2de0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

容器 container
```
可读写的统一文件系统，加上隔离的进程空间，以及其中包含的进程。
可以理解运行时，在只读的镜像层上，套了一层可读写层，从而实现容器的个性化需求。
```
![image.png](https://upload-images.jianshu.io/upload_images/5344773-232ed6c9ed051e3a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![image.png](https://upload-images.jianshu.io/upload_images/5344773-f0a6ca8fa4f2f52d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

联合文件加载系统 UnionFS
```
1. 下载docker镜像的时候，每一层对应一个id，层是AUFS （Advanced UnionFS联合文件系统）中的重要概念;
2. 联合文件系统（UnionFS）是一种分层、轻量级并且高性能的文件系统，它支持对文件系统的修改作为一次提交来一层层的叠加，同时可以将不同目录挂载到同一个虚拟文件系统下;
3. 联合文件系统是 Docker 镜像的基础。镜像可以通过分层来进行继承，基于基础镜像（没有父镜像），可以制作各种具体的应用镜像;
4. 不同 Docker 容器就可以共享一些基础的文件系统层，同时再加上自己独有的改动层，大大提高了存储的效率;
5. Docker 目前支持的联合文件系统种类包括 AUFS, btrfs, vfs 和 DeviceMapper。
```

### 系统加载与Linux区别
#### 传统Linux系统加载
典型的Linux文件系统由 ***bootfs*** 和 ***rootfs***  两部分组成，***bootfs*** (boot file system)主要包含 ***bootloader***和***kernel***，***bootloader***主要是引导加载***kernel***，当***kernel***被加载到内存中后 ***bootfs***就被umount了。 ***rootfs*** (root file system) 包含的就是典型 Linux 系统中的/dev，/proc，/bin，/etc等标准目录和文件。　

传统的Linux加载bootfs时会先将rootfs设为read-only，然后在系统自检之后将rootfs从read-only改为read-write，然后我们就可以在rootfs上进行写和读的操作了。　

#### Docker容器系统加载
Docker容器是建立在Aufs基础上的，Aufs（以前称之为Another Union FS，后来绝不不够高大上，更名为Advanced Union FS）是一种Union FS， 简单来说就是支持将不同的目录挂载到同一个虚拟文件系统下，并实现一种layer的概念。

Aufs将挂载到同一虚拟文件系统下的多个目录分别设置成read-only，read-write以及whiteout-able权限，对read-only目录只能读，而写操作只能实施在read-write目录中。重点在于，写操作是在read-only上的一种增量操作，不影响read-only目录。当挂载目录的时候要严格按照各目录之间的这种增量关系，将被增量操作的目录优先于在它基础上增量操作的目录挂载，待所有目录挂载结束了，继续挂载一个read-write目录，如此便形成了一种层次结构。

Docker的镜像运行时，在bootfs自检完毕之后并不会把rootfs的read-only改为read-write。而是利用union mount（UnionFS的一种挂载机制）将一个或多个read-only的rootfs加载到之前的read-only的rootfs层之上。在加载了这么多层的rootfs之后，仍然让它看起来只像是一个文件系统，在Docker的体系里把union mount的这些read-only的rootfs叫做Docker的镜像。但是，此时的每一层rootfs都是read-only的，我们此时还不能对其进行操作。当我们创建一个容器，也就是将Docker镜像进行实例化，系统会在一层或是多层read-only的rootfs之上分配一层空的read-write的rootfs。

### 常用命令
#### 容器 container
```
【docker search】
#查看指定内容镜像
docker search centos
# NAME                               DESCRIPTION                                     STARS               # OFFICIAL            AUTOMATED
# centos                             The official build of CentOS.                   5387                [# OK]                
# ansible/centos7-ansible            Ansible on Centos7                              # 121                                     [OK]
# jdeathe/centos-ssh                 CentOS-6 6.10 x86_64 / CentOS-7 7.5.1804 x86…   109     

# stars>=100 centos 镜像
docker search --filter=stars=100 centos  

# 查看明细
# docker search --no-trunc centos
# NAME                               DESCRIPTION                                                                                          STARS               OFFICIAL            AUTOMATED
# centos                             The official build of CentOS.                                                                        5387                [OK]                
# ansible/centos7-ansible            Ansible on Centos7                                                                                   121                                     [OK]
# jdeathe/centos-ssh                 CentOS-6 6.10 x86_64 / CentOS-7 7.5.1804 x86_64 - SCL/EPEL/IUS Repos / Supervisor / OpenSSH.         109                                     [OK]

# 自动构建
docker search --filter=is-automated=true centos
# NAME                              DESCRIPTION                                     STARS               OFFICIAL            AUTOMATED
# ansible/centos7-ansible           Ansible on Centos7                              121                                     [OK]
# jdeathe/centos-ssh                CentOS-6 6.10 x86_64 / CentOS-7 7.5.1804 x86…   109                                     [OK]
# consol/centos-xfce-vnc            Centos container with "headless" VNC session…   91                                      [OK]

【docker pull】
# 默认拉取最新镜像
docker pull centos 

# 拉取指定版本镜像
docker pull centos:6.8 
# 6.8: Pulling from library/centos

【docker run】
# 运行容器(-i 交互式, -t 模拟终端,-d 阻塞,--name 容器命名, /bin/sh -c ... 容器执行命名)
docker run -it -d --name mycentos centos:6.8 /bin/sh -c 'while true;do echo `date`; sleep 2 ;done'
# 8f6a378f18678889fb078c711d3120658e0c426ecbbe74a9f0bc1727615afc94

# 查看正在运行容器 (-a 全部容器 --no-trunc)
docker ps
# CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS               NAMES
# 8f6a378f1867        centos:6.8          "/bin/sh -c 'while t…"   3 minutes ago       Up 3 minutes                            mycentos

# 查看容器日志 (-f 滚动监控)
docker logs -f mycentos
# Fri Jun 7 10:40:45 UTC 2019
# Fri Jun 7 10:40:47 UTC 2019
# Fri Jun 7 10:40:49 UTC 2019

# 停止容器
docker stop mycentos

# 删除容器
docker rm mycentos

# 杀死运行容器
docker kill mycentos

# 启动并直接进入bash
docker run -it --name mybash centos:6.8 bash
# [root@44a2f6008195 /]# 
# exit 退出并关闭容器
Ctrl+P, Ctrl+Q 退出并在后台运行容器

# -d 启动并挂在后台运行（否则直接退出）
docker run -it --name mydeamon -d centos:6.8
# 7a1b7349c81e7ae51c32f6ae111c8f1d5380cb32ecd36fcf4e776e48bb94b45a
# docker ps
# CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
# 7a1b7349c81e        centos:6.8          "/bin/bash"         10 seconds ago      Up 9 seconds                            mydeamon

# (-p port1:port2) 启动将容器port2端口映射到宿主机port1端口
docker run -it -p 8080:8080 -d --name mycat tomcat 
# docker ps (0.0.0.0:8080->8080/tcp 端口映射成功)
# CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS                    NAMES
# f76185505e3a        tomcat              "catalina.sh run"   3 seconds ago       Up 2 seconds        0.0.0.0:8080->8080/tcp   mycat
# docker logs -f mycat (tomcat 启动日志)
# 07-Jun-2019 11:39:05.646 INFO [main] org.apache.coyote.AbstractProtocol.start Starting ProtocolHandler ["http-nio-8080"]
# 07-Jun-2019 11:39:05.724 INFO [main] org.apache.coyote.AbstractProtocol.start Starting ProtocolHandler ["ajp-nio-8009"]
# 07-Jun-2019 11:39:05.731 INFO [main] org.apache.catalina.startup.Catalina.start Server startup in 1597 ms

# curl http://docker:8080/  查看端口映射

# (-P)镜像EXPOSE端口随机映射到宿主机
docker run -d -it --name mycat2 -P tomcat     
# d540c1620ecfa2734aa4ff2eee65e6b7ca58d6711d56cb7e08dde9e4c96b2f21
# docker ps
# CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS                     NAMES
# d540c1620ecf        tomcat              "catalina.sh run"   4 seconds ago       Up 2 seconds        0.0.0.0:32768->8080/tcp   mycat2
# f76185505e3a        tomcat              "catalina.sh run"   6 minutes ago       Up 6 minutes        0.0.0.0:8080->8080/tcp    mycat

mkdir /usr/tmp/test
cd /usr/tmp/test
touch 1.txt
# -v 挂在数据卷
docker run -it -d -v /usr/tmp/test:/usr/tmp/test --name myvol  centos:6.8 

# 登入容器(查看数据卷映射，读写权限)
docker attach mycat 
# [root@4374114e4fd1 /]# ls /usr/tmp/test/
# 1.txt
# [root@4374114e4fd1 /]# echo 'hello' >> /usr/tmp/test/1.txt 
# [root@4374114e4fd1 /]# cat /usr/tmp/test/1.txt 
# hello

# 查看容器明细
docker inspect mycat
# {
# 	....
# 	"Mounts": [
# 	            {
# 	                "Type": "bind",
# 	                "Source": "/usr/tmp/test",
# 	                "Destination": "/usr/tmp/test",
# 	                "Mode": "",
# 	                "RW": true,
# 	                "Propagation": "rprivate"
# 	            }
# 	        ],
# 	...
# }

# (-v dir1:dir2:ro) 挂在只读数据卷，默认为rw
docker run -v /usr/tmp/test:/usr/tmp/test:ro -it --name myvol1 centos:6.8 bash
# [root@7025dbea92a1 /]# ls /usr/tmp/test/
# 1.txt
# [root@7025dbea92a1 /]# echo 'hi' >> /usr/tmp/test/1.txt 
# bash: /usr/tmp/test/1.txt: Read-only file system

docker inspect myvol1
# {
# 	...
# 	"Mounts": [
# 	    {
# 	        "Type": "bind",
# 	        "Source": "/usr/tmp/test",
# 	        "Destination": "/usr/tmp/test",
# 	        "Mode": "ro",  <<<
# 	        "RW": false,
# 	        "Propagation": "rprivate"
# 	    }
# 	],
# 	...
# }

# 重新启动容器
docker restart mycat

# 查看正在运行容器
docker ps

# 查看全部容器 (不截断信息)
docker ps -a --no-trunc
```

#### 镜像 image
```
# 查找镜像
docker search tomcat

# 查找指定版本镜像
docker search tomcat:9.0
# NAME                DESCRIPTION                           STARS               OFFICIAL            AUTOMATED
# iyayu/tomcat        Java JDK:1.8.0_131 Tomcat:9.0.0.M22   0  

# 查找评分在100以上镜像
docker search --filter=stars=100 tomcat
# NAME                DESCRIPTION                                     STARS               OFFICIAL            AUTOMATED
# tomcat              Apache Tomcat is an open source implementati…   2410                [OK]  

# 查找自动构建镜像
docker search --filter=is-automated=true tomcat
# NAME                          DESCRIPTION                                     STARS               OFFICIAL            AUTOMATED
# dordoka/tomcat                Ubuntu 14.04, Oracle JDK 8 and Tomcat 8 base…   53                                      [OK]

# 拉取指定版本镜像
docker pull tomcat:8.5
# 8.5: Pulling from library/tomcat
# Digest: sha256:d4a94787b1124f4269180e74e6c29ec9512bdb8c772c8aa1c7e7e822ce633618
# Status: Downloaded newer image for tomcat:8.5

# 将容器转换为本地镜像(-a author, -m message)
docker commit -a 'xx<xxx@xxx.com>' -m 'tomcat:9.0 in centos6.8' d540c1620ecf tomcat/centos6.8:v9.0
# sha256:1da2dff72b71c0cd45e652aa4be28e0e1510929708fb05962300d7b99aac12f7
# docker images
# REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
# tomcat/centos6.8    v9.0                1da2dff72b71        3 seconds ago       522MB

# 容器提交为本地镜像（忘记打标签）
docker commit -a 'xx<xx@163.com>' -m 'tomcat:9.0 on centos6.8' d540c1620ecf 
# sha256:66797a68a41b951c678f663443dccdf5033122aa2365677abc7cb7114995384c

# 查看全部镜像
docker images
# REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
# <none>              <none>              66797a68a41b        4 seconds ago       522MB

# 补标签
 docker tag 66797a68a41b tomact/centos6.8:v9.0
# docker images
# REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
# tomact/centos6.8    v9.0                66797a68a41b        33 seconds ago      522MB

mkdir  /usr/tmp/dockerfile

# 创建DockerFile文件
vim /usr/tmp/dockerfile/DockerFile
# -----------------------------------
# # centos6.8 with hello-world
# FROM centos:6.8
# MAINTAINER xx<xx@xx.com>
# CMD echo 'hello world'
# -----------------------------------

# 基于DockerFile构建镜像
docker build -f /usr/tmp/dockerfile/DockerFile -t myhello/centos6.8:v1.0  /usr/tmp/dockerfile/
# Sending build context to Docker daemon  2.048kB
# Step 1/3 : FROM centos:6.8
#  ---> 82f3b5f3c58f
# Step 2/3 : MAINTAINER xx<xx@xx.com>
#  ---> Running in 1ce6df6a79e4
# Removing intermediate container 1ce6df6a79e4
#  ---> 4300aac5726a
# Step 3/3 : CMD echo 'hello world'
#  ---> Running in e2b443c58df4
# Removing intermediate container e2b443c58df4
#  ---> 326e32a3cb3d
# Successfully built 326e32a3cb3d
# Successfully tagged myhello/centos6.8:v1.0

# docker images
# REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
# myhello/centos6.8   v1.0                326e32a3cb3d        2 minutes ago       195MB

# 运行自定义镜像
docker run --name hello myhello/centos6.8:v1.0
# hello world
```

#### 仓库 Repo
https://cr.console.aliyun.com/cn-hangzhou/instances/repositories
1. 创建命名空间 centos68-registry
2. 创建镜像仓库 tomcat [从本地镜像推送]
3. 镜像仓库 > 管理
```
 docker login --username=xxx registry.cn-hangzhou.aliyuncs.com
# Password: 
# WARNING! Your password will be stored unencrypted in /root/.docker/config.json.
# Configure a credential helper to remove this warning. See
# https://docs.docker.com/engine/reference/commandline/login/#credentials-store
# 
# Login Succeeded

docker images 
# REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
# tomact/centos6.8    v9.0                66797a68a41b        37 minutes ago      522MB  << 准备push的镜像

# 标注远程仓库tag
docker tag  66797a68a41b registry.cn-hangzhou.aliyuncs.com/centos68-registry/tomcat:v1.0
# docker images 
# REPOSITORY                                                   TAG                 IMAGE ID            CREATED             SIZE
# registry.cn-hangzhou.aliyuncs.com/centos68-registry/tomcat   v1.0                66797a68a41b        41 minutes ago      522MB
# tomact/centos6.8                                             v9.0                66797a68a41b        About an hour ago   522MB

# 推送远程仓库
docker push registry.cn-hangzhou.aliyuncs.com/centos68-registry/tomcat:v1.0
# The push refers to repository [registry.cn-hangzhou.aliyuncs.com/centos68-registry/tomcat]
# 8ca06e1b99fd: Pushed 
# .....
# 95c8d3f0ae82: Pushing [======================>                            ]  90.71MB/204.8MB
# a3e9c8cfc64e: Pushed
# v1.0: digest: sha256:f41aa4f8da6775d8030c70dbd2517bfa7311c0386a2bbdd346ba786b3bbe4fb3 size: 3051 

# 删除标签 和 本地镜像
docker rmi registry.cn-hangzhou.aliyuncs.com/centos68-registry/tomcat:v1.0 tomact/centos6.8:v9.0

# 远程拉取镜像
docker pull  registry.cn-hangzhou.aliyuncs.com/centos68-registry/tomcat:v1.0

# 查看指定仓库相关镜像
docker images docker images registry.cn-hangzhou.aliyuncs.com/centos68-registry/tomcat
# REPOSITORY                                                   TAG                 IMAGE ID            CREATED             SIZE
# registry.cn-hangzhou.aliyuncs.com/centos68-registry/tomcat   v1.0                66797a68a41b        About an hour ago   522MB

# 运行
docker run -d -it --name mycat2 registry.cn-hangzhou.aliyuncs.com/centos68-registry/tomcat:v1.0

# 查看正在运行容器ID
docker ps -q
# d949db870969
# 7025dbea92a1
# 4374114e4fd1
# 7a1b7349c81e

# 关闭所有正在运行容器
docker stop $(docker ps -q)

```
### DockerFile语法
#### 示例
```
vim /usr/tmp/dockerfile/DockerFile_jdk_base
# ----------------------------------------------------
# 基础镜像来源 centos6.8 java-1.8.0-openjdk-devel.x86_64
FROM centos:6.8

# 维护人
MAINTAINER xx<xx@xx.com>

# 可以在docker build 阶段被 --build-arg 覆盖，构建时与ENV一致，运行时只有ENV可以转换为环境
ARG SOFT=/opt/software

# 声明环境变量，
ENV BASE $SOFT

# 创建目录
VOLUME [$SOFT]

# 执行命令
# RUN yum install -y java-1.8.0-openjdk-devel.x86_64
ADD jdk-8u141-linux-x64.tar.gz $BASE 

# 作为子类被引用时，才会构建，需要引用ENV，而不是ARG
ONBUILD ENV JAVA_HOME $BASE/jdk1.8.0_141
ONBUILD ENV CLASSPATH .:$JAVA_HOME/jre/lib/rt.jar:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
ONBUILD ENV PATH $PATH:$JAVA_HOME/bin

# END
# ----------------------------------------------------

# 构建带jdk基础镜像
docker build -f DockerFile_jdk_base -t jdk/centos6.8:1.8 .
Sending build context to Docker daemon  196.4MB
Step 1/9 : FROM centos:6.8
 ---> 82f3b5f3c58f
Step 2/9 : MAINTAINER xx<xx@xx.com>
 ---> Running in 30d590e4f0ae
Removing intermediate container 30d590e4f0ae
 ---> 4ca10c4bfcbe
Step 3/9 : ARG SOFT=/opt/software
 ---> Running in 7df4c86e95ff
Removing intermediate container 7df4c86e95ff
 ---> df7b1d2a36f2
Step 4/9 : ENV BASE $SOFT
 ---> Running in b6ee40626fc0
Removing intermediate container b6ee40626fc0
 ---> 7e6e5779631e
Step 5/9 : VOLUME [$SOFT]
 ---> Running in 6c3bc11a72ea
Removing intermediate container 6c3bc11a72ea
 ---> 6a6d801fec11
Step 6/9 : ADD jdk-8u141-linux-x64.tar.gz $BASE
 ---> d66d86e17e88
Step 7/9 : ONBUILD ENV JAVA_HOME $BASE/jdk1.8.0_141
 ---> Running in f1250da248be
Removing intermediate container f1250da248be
 ---> ddf0fee19b28
Step 8/9 : ONBUILD ENV CLASSPATH .:$JAVA_HOME/jre/lib/rt.jar:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
 ---> Running in 17cde3c07573
Removing intermediate container 17cde3c07573
 ---> 440385088d3d
Step 9/9 : ONBUILD ENV PATH $PATH:$JAVA_HOME/bin
 ---> Running in c5955c31bf3c
Removing intermediate container c5955c31bf3c
 ---> ed0710291179
Successfully built ed0710291179
Successfully tagged jdk/centos6.8:1.8
vim /usr/tmp/dockerfile/DockerFile_tomcat_enhanced

# 配置可远程访问管理的tomcat
vim server.xml
# ----------------------------------------------------
   <Connector port="8080" protocol="HTTP/1.1"
               connectionTimeout="20000"
               redirectPort="8443"
               URIEncoding="UTF-8" />
<!-- 修改默认编码-->
# ----------------------------------------------------

# 基于角色授权
vim tomcat-users.xml 
# ----------------------------------------------------
<role rolename="manager-gui"/>
<role rolename="manager-script"/>
<role rolename="manager-jmx"/>
<role rolename="manager-status"/>
<role rolename="admin-gui"/>
<role rolename="admin-script"/>
<user username="tomcat" password="tomcat" roles="manager-gui,manager-script,manager-jmx,manager-status,admin-gui,admin-script"/>
# ----------------------------------------------------

# 注释掉访问IP限制
vim context.xml 
# ----------------------------------------------------
<Context antiResourceLocking="false" privileged="true" >
  <!--
  <Valve className="org.apache.catalina.valves.RemoteAddrValve"
         allow="127\.\d+\.\d+\.\d+|::1|0:0:0:0:0:0:0:1" />
  -->
  <Manager sessionAttributeValueClassNameFilter="java\.lang\.(?:Boolean|Integer|Long|Number|String)|org\.apache\.catalina\.filters\.CsrfPreventionFilter\$LruCache(?:\$1)?|java\.util\.(?:Linked)?HashMap"/>
</Context>
# ----------------------------------------------------

# 添加jsp
# vim /opt/docker_house/tomcat9.0/test/hello.jsp
# ----------------------------------------------------
<%@ page contentType="text/html;charset=UTF-8" language="java"%>

<!DOCTYPE html>

<html>
<head>
<title>why are you not working</title>
<meta charset="utf-8" />
</head>

   <body>
      <h1>hello</h1>
       <%
            System.out.println("hello");
       %>
   </body>
</html>
# ----------------------------------------------------


vim DockerFile_tomcat
# ----------------------------------------------------
# centos6.8 jdk1.8.0 + apache-tomcat-9.0.20(enhanced)
# 继承
FROM jdk/centos6.8:1.8

# 维护人
MAINTAINER xx<xx@xx.com>

# docker build -d xx -t xx:yy --build-arg BASE=/opt/software 构建参数(此处为安装路径)
# 此处赋予默认值(也可以不赋值)
ARG BASE=/opt/software

# 挂载容器目录，相当于新建目录
VOLUME [$BASE]

# 拷贝并解压宿主机压缩文件到指定容器路径
ADD apache-tomcat-9.0.20.tar.gz $BASE

# 声明DockerFile环境变量
ENV CATALINA_HOME $BASE/apache-tomcat-9.0.20
ENV CATALINA_BASE $CATALINA_HOME
ENV CATALINA_TMPDIR $CATALINA_HOME/temp
ENV PATH $PATH:$CATALINA_HOME/bin

# 删除原生配置（以下是替换策略，还可以使用 -v 添加数据卷方式，动态从宿主机映射配置文件到容器）
RUN rm -rf $CATALINA_HOME/conf/server.xml
RUN rm -rf $CATALINA_HOME/webapps/manager/META-INF/context.xml
RUN rm -rf $CATALINA_HOME/conf/tomcat-users.xml

# 纯拷贝文件，绝对路径代表镜像内部路径，相对路径代表当前(宿主机)构建路径
# utf-8 乱码
COPY ./server.xml $CATALINA_HOME/conf
# 权限
COPY ./tomcat-users.xml $CATALINA_HOME/conf
# 远程访问ip限制
COPY ./context.xml  $CATALINA_HOME/webapps/manager/META-INF/

# 声明工作目录
WORKDIR $CATALINA_HOME

# 声明容器暴露端口 (-P 启动时随机映射， 8080 web端口, 8000 jpda 远程调试端口)
EXPOSE 8080 8000

# 两种默认启动方式二选一，CMD可被启动命令覆盖，ENTRYPOINT接受启动传参
ENTRYPOINT $CATALINA_HOME/bin/catalina.sh run
# CMD $CATALINA_HOME/bin/catalina.sh run && tail -F $CATALINA_HOME/logs/catalina.out

# END
# ----------------------------------------------------

 docker images
REPOSITORY                                                   TAG                 IMAGE ID            CREATED             SIZE
jdk/centos6.8                                                1.8                 ed0710291179        3 minutes ago       571MB

# 构建tomcat镜像
docker build -f DockerFile_tomcat -t tomcat/centos6.8_jdk1.8:9.0 . 
Sending build context to Docker daemon  196.4MB
Step 1/18 : FROM jdk/centos6.8:1.8
# Executing 3 build triggers  《《《《 ONBUILD
 ---> Running in aef51e50ae84
Removing intermediate container aef51e50ae84
 ---> Running in 7a7b1906bf42
Removing intermediate container 7a7b1906bf42
 ---> Running in 5f7bafdbed25
Removing intermediate container 5f7bafdbed25
 ---> e937abb3548b
Step 2/18 : MAINTAINER xx<xx@xx.com>
 ---> Running in 1a222aea98c7
Removing intermediate container 1a222aea98c7
 ---> 8c25ea47c074
Step 3/18 : ARG BASE=/opt/software
 ---> Running in d11c760bf803
Removing intermediate container d11c760bf803
 ---> dd3e94b14a9f
Step 4/18 : VOLUME [$BASE]
 ---> Running in 246cf38c70f9
Removing intermediate container 246cf38c70f9
 ---> 0e1dd4fb9bf4
Step 5/18 : ADD apache-tomcat-9.0.20.tar.gz $BASE
 ---> e8e8bfc9ea6b
Step 6/18 : ENV CATALINA_HOME $BASE/apache-tomcat-9.0.20
 ---> Running in ef7e2d1ecf39
Removing intermediate container ef7e2d1ecf39
 ---> 34f7c45bddfb
Step 7/18 : ENV CATALINA_BASE $CATALINA_HOME
 ---> Running in 8d14e2004d00
Removing intermediate container 8d14e2004d00
 ---> f1a21e8a4659
Step 8/18 : ENV CATALINA_TMPDIR $CATALINA_HOME/temp
 ---> Running in 754085cb3755
Removing intermediate container 754085cb3755
 ---> a47e6dac4987
Step 9/18 : ENV PATH $PATH:$CATALINA_HOME/bin
 ---> Running in 2205784c95f3
Removing intermediate container 2205784c95f3
 ---> ed2a20445b00
Step 10/18 : RUN rm -rf $CATALINA_HOME/conf/server.xml
 ---> Running in 75f19c512bd2
Removing intermediate container 75f19c512bd2
 ---> 9f07131f1f8f
Step 11/18 : RUN rm -rf $CATALINA_HOME/webapps/manager/META-INF/context.xml
 ---> Running in a00638b3fa19
Removing intermediate container a00638b3fa19
 ---> 878eaf80dbbc
Step 12/18 : RUN rm -rf $CATALINA_HOME/conf/tomcat-users.xml
 ---> Running in cdf421b5a2d5
Removing intermediate container cdf421b5a2d5
 ---> 4be8ec1e835e
Step 13/18 : COPY ./server.xml $CATALINA_HOME/conf
 ---> 931e76498b14
Step 14/18 : COPY ./tomcat-users.xml $CATALINA_HOME/conf
 ---> 0a169c3155e7
Step 15/18 : COPY ./context.xml  $CATALINA_HOME/webapps/manager/META-INF/
 ---> d704b5269238
Step 16/18 : WORKDIR $CATALINA_HOME
 ---> Running in efe8e65ca7a1
Removing intermediate container efe8e65ca7a1
 ---> bedc960a2690
Step 17/18 : EXPOSE 8080 8000
 ---> Running in fdd5bb86c840
Removing intermediate container fdd5bb86c840
 ---> 531731b08c83
Step 18/18 : ENTRYPOINT $CATALINA_HOME/bin/catalina.sh run
 ---> Running in 653cf12510d9
Removing intermediate container 653cf12510d9
 ---> 64cd67aeeadc
Successfully built 64cd67aeeadc
Successfully tagged tomcat/centos6.8_jdk1.8:9.0

# 查看镜像
docker images
REPOSITORY                                                   TAG                 IMAGE ID            CREATED             SIZE
tomcat/centos6.8_jdk1.8                                      9.0                 64cd67aeeadc        57 seconds ago      586MB
jdk/centos6.8                                                1.8                 ed0710291179        3 minutes ago       571MB

# 运行容器 （-it交互式伪终端，-d守护式，-p端口映射，-v 路径映射）
docker run -it -d -p 8080:8080 -p 8000:8000 \
-v /opt/docker_house/tomcat9.0/logs:/opt/software/apache-tomcat-9.0.20/logs \
-v /opt/docker_house/tomcat9.0/test:/opt/software/apache-tomcat-9.0.20/webapps/test \
--name cat tomcat/centos6.8_jdk1.8:9.0 

# 查看正在运行容器
docker ps
CONTAINER ID        IMAGE                         COMMAND                  CREATED             STATUS              PORTS                                            NAMES
dd55a77a31d2        tomcat/centos6.8_jdk1.8:9.0   "/bin/sh -c '$CATALI…"   3 seconds ago       Up 1 second         0.0.0.0:8000->8000/tcp, 0.0.0.0:8080->8080/tcp   cat

# -p -P 端口映射参数，必须开启防火墙
# 允许开机启动防火墙
systemctl enable firewalld

# 打开防火墙
systemctl start firewalld

# 永久开放8080端口
firewall-cmd --zone=public --add-port=8080/tcp --permanent

#重启防火墙
firewall-cmd --reload

# 查看全部打开端口
firewall-cmd --list-all
 
# 查看远程访问
http://docker:8080/

# 本地通过挂载目录查看日志
tail -f /opt/docker_house/tomcat9.0/logs/* 

# 查看动态部署
http://docker:8080/test/hello.jsp

# 基于运行容器构建”黑箱镜像(不明确构建过程的镜像)“ (-a 作者，-m提交信息)
docker commit -a whohow20094702 -m 'tomcat_enhanced' cat tomcat/centos6.8_jdk1.8:9.1

docker images
REPOSITORY                                                   TAG                 IMAGE ID            CREATED             SIZE
tomcat/centos6.8_jdk1.8                                      9.1                 2576602400f3 <      6 seconds ago       586MB
tomcat/centos6.8_jdk1.8                                      9.0                 64cd67aeeadc        3 hours ago         586MB
jdk/centos6.8                                                1.8                 ed0710291179        3 hours ago         571MB

# 登录注册仓库
docker login --username=16601040128 registry.cn-hangzhou.aliyuncs.com
#Password: a..8
#Login Succeeded

# 打标签
docker tag 2576602400f3 registry.cn-hangzhou.aliyuncs.com/centos68-registry/tomcat:9.1

# 远程推送
docker push registry.cn-hangzhou.aliyuncs.com/centos68-registry/tomcat:9.1
# The push refers to repository [registry.cn-hangzhou.aliyuncs.com/centos68-registry/tomcat]
# 6fee83e9a456: Pushed 
# 7530db3afc8c: Pushed 
# 3f6336fe2b37: Pushed 
# 04fe25096745: Pushed 
# c20b415d3cf3: Pushed 
# c8aec80bb90b: Pushed 
# 3acad8d0388c: Pushed 
# ec66544a95eb: Pushed 
# 0cc8fd615a43: Pushing [=====================================>             ]  280.7MB/376.2MB
# ad337ac82f03: Mounted from centos7-registry/docker-test 
9.1: digest: sha256:e16564ced2f986d431316027291a33aede23c591b218ac55cefd81d0edbf8784 size: 2408

# 从远程仓库拉取
docker pull registry.cn-hangzhou.aliyuncs.com/centos68-registry/tomcat:9.1
```






