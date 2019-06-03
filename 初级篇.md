## 关键词
应用场景 加速器 安装 删除
## 是什么？
    1. Docker 是一个开源的应用容器引擎，基于 Go 语言 并遵从Apache2.0协议开源。
    2. Docker 可以让开发者打包他们的应用以及依赖包到一个轻量级、可移植的容器中，然后发布到任何流行的 Linux 机器上，也可以实现虚拟化。
    3. 容器是完全使用沙箱机制，相互之间不会有任何接口（类似 iPhone 的 app）,更重要的是容器性能开销极低。从17.03开始分为社区版（CE）和 企业版（EE）。
## 为什么要用？
    1. 简化程序(方便快捷带环境交付)；
    2. 快速实践,持续集成(避免选择焦虑)；
    3. 支持水平伸缩，快速扩容，节省开支。
## 哪些场景使用？
    1. 避免开发、部署环境差异导致软件交付争议；
    2. Web 应用的自动化打包和发布;
    3. 自动化测试和持续集成、发布;
    4. 在服务型环境中部署和调整数据库或其他的后台应用;
    5. 从头编译或者扩展现有的OpenShift或Cloud Foundry平台来搭建自己的PaaS环境。
## 如何使用？
### 安装
#### centos 6.5(64位)

#### centos 7.x(64位)
```
# 核对内核版本
uname -r
3.10.0-957.5.1.el7.x86_64

cat /etc/redhat-release 
CentOS Linux release 7.6.1810 (Core) 

# 移出旧版
udo yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-selinux \
                  docker-engine-selinux \
                  docker-engine

# 安装必要系统工具
sudo yum install -y yum-utils device-mapper-persistent-data lvm2

# 添加 yum 镜像源
sudo yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

# 更新缓存
sudo yum makecache fast

# 安装社区版
sudo yum -y install docker-ce

# 启动守护进程
sudo systemctl start docker

# 查看
docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
```
### 云镜像加速
```
# 阿里云加速
# https://cr.console.aliyun.com/cn-hangzhou/instances/mirrors
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://phwu2j1l.mirror.aliyuncs.com"]
}

# 网易云加速
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["http://hub-mirror.c.163.com"]
  "max-concurrent-downloads": 10
}
```
### 测试
```
docker pull hello-world
# Unable to find image 'hello-world:latest' locally (优先从本地找)
# latest: Pulling from library/hello-world (然后从远程镜像仓库拉)
# Digest: sha256:0e11c388b664df8a27a901dce21eb89f11d8292f7fca1b3e3c4321bf7897bffe
# Status: Downloaded newer image for hello-world:latest

# Hello from Docker!   (成功运行)
# This message shows that your installation appears to be working correctly.

# To generate this message, Docker took the following steps:
 # 1. The Docker client contacted the Docker daemon.
 # 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
 #    (amd64)
 # 3. The Docker daemon created a new container from that image which runs the
  #   executable that produces the output you are currently reading.
 # 4. The Docker daemon streamed that output to the Docker client, which sent it
  #   to your terminal.

# To try something more ambitious, you can run an Ubuntu container
# with:
#  $ docker run -it ubuntu bash

# Share images, automate workflows, and more with a free Docker ID:
#  https://hub.docker.com/

# For more examples and ideas, visit:
#  https://docs.docker.com/get-started/
```
### 删除
```
sudo yum remove docker-ce
sudo rm -rf /var/lib/docker
```
