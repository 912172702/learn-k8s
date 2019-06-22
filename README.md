# learn-k8s

## 一、Docker 学习

### 1.1 Docker的理解

Docker是一种虚拟化技术，它比虚拟机更轻量，不考虑考虑硬件设备的虚拟化，只提供需要的软件依赖。所以比虚拟机更加简便、容易移植。开发人员将软件产品和它的所有依赖库都打包到一个Docker镜像中，交给运维，大大减少了运维的工作量。因此开发可以代替运维，萌生出了一种新的职业，DepOps，开发/运维 工程师。

### 1.2 Docker的整体架构 

![AD272-A13-6-D9-B-4511-987-D-6119-C23399-A0.jpg](https://i.postimg.cc/j2fFqbzN/AD272-A13-6-D9-B-4511-987-D-6119-C23399-A0.jpg)

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

[![7841-CA50-CB33-4-EE0-A876-F05541-E15-AC4.jpg](https://i.postimg.cc/mkhM9vBr/7841-CA50-CB33-4-EE0-A876-F05541-E15-AC4.jpg)](https://postimg.cc/xNDcDFkr)

