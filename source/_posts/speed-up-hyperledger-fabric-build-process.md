---
title: 加速Hyperledger Fabric的docker镜像构建过程
date: 2020-08-18 16:26:40
tags: [docker, hyperledger-fabric]
categories: [blockchain]
---

Hyperledger Fabric 从`v2.0`开始，全面将docker基础镜像替换成了体积更小、潜在安全风险更少、更加轻量的`Alpine Linux`，从而使得`make docker`出来的各种镜像的体积几乎都缩小为原来的一半，确实能够节省更多的硬盘空间。但是，由于众所周知的原因，对于生在红旗下，长在新中国的程序员们，第一次在Fabric项目下构建docker镜像时，依然是奇慢无比，屡次超时。

那么这个问题怎么解决呢？

## 分析速度的瓶颈

首先通过分析速度慢的原因，找出可以优化的点。通过分析`make docker`命令，大概过程是这样的：首先是docker会从docker registry pull Alpine作为基础镜像，然后使用`apk add --no-cache xxx`安装一些软件，最后通过make命令build出Peer、Order以及其他的tools的二进制包。到这里相信国内的各种奇人义士已经磨刀霍霍，迫不及待的开始替换各种国内的mirror了。
<!-- more -->
下面的各种资源都来源于网上各位好心人的分享。

## 加速docker pull过程

首先从网上搜索国内的docker registry源，然后修改docker的配置并**重启**docker。在这里我比较推荐使用自己专有的免费的阿里云镜像加速器，目前使用一直比较平稳。

* 获取镜像加速url

注册一个阿里云账号并登录，在`产品与服务`中搜索`容器镜像服务`，跟随引导完成必要的一些步骤，然后来到这个页面：https://cr.console.aliyun.com/cn-hangzhou/instances/mirrors，就可以看到自己专有的加速器地址了。

* 给docker客户端配置镜像加速器

如果没有`/etc/docker/daemon.json`文件，则可以直接通过下面的命令完成配置；如果该文件已经存在了，则选择自己熟悉的文本编辑器编辑该文件，添加加速器地址的配置。如果你使用的带界面的客户端，也可以在`Preferences... -> Docker Engine`显示的编辑器中添加相应的配置。

```shell
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://xxx.mirror.aliyuncs.com"]
}
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker

```

**注意**：将`https://xxx.mirror.aliyuncs.com`替换成你自己的镜像加速器地址。再次提醒，配置完成后，**重启**docker之后才会生效。

至此我们完成了docker pull镜像阶段的加速，接下来我们加速在Alpine中安装软件的过程。

## 加速Alpine安装软件的过程

在docker pull下来的Alpine镜像中，使用apk安装软件时默认使用的是国外的源，速度比较慢。这里我们把源替换为国内的源，可以大大节省时间。下面以Hyperledger fabric peer的[Dockerfile](https://github.com/hyperledger/fabric/blob/master/images/peer/Dockerfile)为例，替换为阿里的源：

```dockerfile
...

FROM alpine:${ALPINE_VER} as peer-base
# 使用下面这行命令完成源的替换
RUN sed -i 's/dl-cdn.alpinelinux.org/mirrors.aliyun.com/g' /etc/apk/repositories
RUN apk add --no-cache tzdata

FROM golang:${GO_VER}-alpine${ALPINE_VER} as golang
# 使用下面这行命令完成源的替换
RUN sed -i 's/dl-cdn.alpinelinux.org/mirrors.aliyun.com/g' /etc/apk/repositories
RUN apk add --no-cache \
	bash \
	gcc \
	git \
	make \
	musl-dev
ADD . $GOPATH/src/github.com/hyperledger/fabric

...
```

另外也可以替换为中科大的源：dl-cdn.alpinelinux.org/mirrors.ustc.edu.cn，或者清华的源：dl-cdn.alpinelinux.org/mirrors.tuna.tsinghua.edu.cn，或者其他的源。

**注意**：这里对于使用multi-stage的Dockerfile，如果不同的stage使用的是不同的基础镜像，则都需要替换源。

到这里，当docker使用Alpine作为基础镜像时，安装依赖软件的过程就会快很多。

