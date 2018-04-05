---
layout: post
title: 基于Harbor搭建docker registry
tags:
- docker
categories: docker
description: 基于Harbor搭建docker registry
---


本文记录一下在Centos7.3操作系统上，基于Harbor来搭建docker registry。当前环境为：
<pre>
# cat /etc/centos-release
CentOS Linux release 7.3.1611 (Core) 

# uname -a
Linux bogon 3.10.0-514.el7.x86_64 #1 SMP Tue Nov 22 16:42:41 UTC 2016 x86_64 x86_64 x86_64 GNU/Linux

# docker --version
Docker version 17.12.1-ce, build 7390fc6

# docker-compose --version
docker-compose version 1.20.1, build 5d8c71b
</pre>


<!-- more -->

## 1. Harbor简介
Harbor工程是一个企业级的镜像服务器，用于存储和分发Docker镜像。Harbor扩展了开源软件```Docker Distribution```，添加了如```security```、```identity```和```management```等功能。作为一个企业级的私有镜像仓库，Harbor提供了更好的性能和安全性。Harbor支持建立多个```registries```,并提供这些仓库间镜像的复制能力。Harbor也提供了更加先进的安全特性，比如用户管理、访问控制、活动审计。

Harbor特性：

* **基于角色的访问控制**: ```users```和```repositories ```都是以projects的方式组织的。在一个project下面，每一个用户对镜像有不同的全向。

* **基于策略的镜像复制**: 在多个registry之间镜像可以同步，并且在出现错误的时候可以进行自动重试。在负载均衡、高可用性、多数据中心和异构云环境下都表现出色。

* **脆弱性扫描(Vulnerability Scanning)**: Harbor会周期性的扫描镜像，然后警告用户相应的脆弱性

* **LDAP/AD支持**: Harbor可以和已存在的企业版LDAP/AD系统集成，以提供用户认证和管理

* **镜像删除 & 垃圾回收**: Images可以被删除，然后回收它们所占用的空间

* **可信任(Notary)**: 可以确保镜像的真实性

* **用户界面(Graphical user portal)**: 用户可以人容易的浏览、搜索仓库和管理工程

* **审计(Auditing)**: 所有对仓库的操作都会被跟踪记录

* **RESTful API**: 对于大部分的管理操作都提供了RESTful API， 很容易和外部系统进行集成

* **易部署**: 提供了离线和在线安装

## 2. Harbor的安装

这里介绍的是通过```Harbor安装文件```的方式来安装Harbor。在Linux操作系统上至少需要如下环境：
<pre>
docker 1.10.0+ and docker-compose 1.6.0+
</pre>

### 2.1 下载Harbor离线安装包
到[Harbor Release](https://github.com/vmware/harbor/releases)页面下载对应的离线安装包，目前我们下载最新版本```v1.4.0```:
<pre>
# mkdir /opt/harbor-inst
# cd /opt/harbor-inst

# wget https://storage.googleapis.com/harbor-releases/release-1.4.0/harbor-offline-installer-v1.4.0.tgz
</pre>

### 2.2 目标主机相关配置推荐

Harbor部署完后会运行多个Docker containers，因此可以部署在任何支持docker的Linux发布版本上。部署的目标主机需要安装```Python```, ```Docker```和```Docker Compose```。

硬件环境：



<br />
<br />

**[参考]**

1. [harbor官网](https://github.com/vmware/harbor)

2. [Centos7上Docker仓库Harbor的搭建](https://blog.csdn.net/felix_yujing/article/details/54694294)
<br />
<br />
<br />
