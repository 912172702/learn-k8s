# learn-k8s

## 一、Docker 学习

### 1.1 Docker的理解

Docker是一种虚拟化技术，它比虚拟机更轻量，不考虑考虑硬件设备的虚拟化，只提供需要的软件依赖。所以比虚拟机更加简便、容易移植。开发人员将软件产品和它的所有依赖库都打包到一个Docker镜像中，交给运维，大大减少了运维的工作量。因此开发可以代替运维，萌生出了一种新的职业，DepOps，开发/运维 工程师。

### 1.2 Docker的整体架构 

![AD272-A13-6-D9-B-4511-987-D-6119-C23399-A0.jpg](https://t1.picb.cc/uploads/2019/06/23/gceSXi.jpg)

#### 1.2.1 镜像和仓库 

- 镜像是一个模版，用来创建Docker容器，镜像(image)和容器(container)的关系类似于Java中的类和对象的关系
- 仓库(Repository)是存放镜像的场所
- 仓库注册服务器(Registry), 仓库注册服务器上有多个仓库。
- 仓库分为公开库会让私有库，最大的公开库是Docker Hub，国内的有阿里云等。

### 1.3 Docker的安装

CentOS使用yum安装Docker。

[点击这里查看安装教程](https://docs.docker.com/install/linux/docker-ce/centos/)

### 1.4 阿里云镜像加速

[阿里云镜像配置](https://cr.console.aliyun.com/cn-hangzhou/instances/mirrors)

### 1.5 Docker的常用命令

#### 1.5.1 docker run

```sh
docker run [镜像名]
```

先在本机找有没有这个镜像，如果没有，就去docker的仓库中拉取这个镜像并运行。过程如下。

![流程](https://t1.picb.cc/uploads/2019/06/23/gce05v.jpg)

#### 1.5.2 docker version

查看docker版本

#### 1.5.3 docker info

docker 信息

#### 1.5.4 docker --help

帮助文档

#### 1.5.5 docker images

列出本地所有docker的镜像

`docker images -q` 列出所有docker镜像的id

`docker images -digests` 显示DIGEST信息，完整ID

#### 1.5.6 docker search

`docker search tomcat` 查看所有tomcat镜像

`docker search -s 30 tomcat` 查看所有点赞数大于等于30的tomcat镜像

#### 1.5.7 docker rmi 

`docker rmi -f [镜像名]`删除单个镜像

`docker rmi -f 镜像名1 镜像名2`删除多个镜像

#### 1.5.8 docker run 

启动一个docker容器

我们拉下里一个centos镜像

```sh
docker pull centos
```

然后启动它

```sh
docker run -i -t centos 
```

`-i`的作用是以交互式的方式启动，`-t`是启动一个tty，terminal，这时就登录了docker内的centos

后面再加 `-name 容器名`，则容器的名字即是指定的名字

`docker ps` 可以查看当前正在运行的容器

`exit` 可以关闭并退出一个容器， `Ctrl + p + q` 退出但不关闭Docker容器

`docker run -d ...`以守护进程的方式运行在后台，如果在命令行直接用这个命令启动一个镜像的话，会导致这个Container启动后立即退出，这个是Docker的一个运行机制，Docker如果以守护进程方式启动，如果无事可做，就会自动退出。不想让它退出就要给他一件事情做。

```sh
docker run -d centos /bin/sh -c "while true;do echo hello caohui ;sleep 2; done"
```

上面这个命令让docker 后台启动，并且每两秒打印出一句话，这时候，就不会推出了。如果想要查看Docker容器内部的打印情况，用 `docker logs -t -f [容器id]`

-t表示显示时间，-f表示实时获取。

#### 1.5.8 docker exec

在Docker外部执行命令控制docker

```sh
docker exec -it mycentos ls #交互式的打印mycentos容器内的文件目录
docker exec -it mycentos /bin/bash #交互式打开mycentos容器中的bash，并进入操作。
```
