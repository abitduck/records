---
layout: post
title: IM服务器设计--网关接入层(转)
tags:
- 分布式系统
categories: distribute-systems
description: IM服务器设计
---

网关接入层负责维护与客户端之间的长连接，由于它是唯一一个与客户端进行直接通信的服务入口，维护着大量的客户端连接，其设计原则应该满足：

* 安全

* 稳定

* 快速

具体来说，需要考虑不少的问题：

* 用什么数据结构保存与客户端的连接？

* 如何清除死链

* 在网关宕机的情况下如何容错

* 服务如何降级？

以下具体展开。


>说明： 工作多年，自己也亲身参与过一款IM系统相关模块的设计，但是对比本文有一些地方还是略有不足。本文转载自[IM服务器设计-基础](https://www.codedump.info/post/20190608-im-design-base/)，主要是为了进一步从更高层次理解IM；另一方面也方便自己的后续查找，防止文章丢失。

<!-- more -->

## 1. 基础设计
简而言之，网关内部维护着一个map,其中保存着客户端相关的ID与对应连接的映射关系。

![im-map](https://ivanzz1001.github.io/records/assets/img/distribute/im/map.png)

内部服务需要应答客户端时，经历如下步骤：

* 到redis中查询路由信息，即客户端连接到了哪个网关，将消息发送给该网关

* 网关服务在上面的map中找到对应的客户端连接，将消息发送给客户端

## 2. 死链的处理
由于网关上维护着大量的客户端连接，需要通过收发心跳包的方式来检查死链，具体做法是：

* 网关针对每个连上的连接，都创建一个定时器

* 网关跟客户端的每次交互之后，网关都对应的更新一下该客户端的心跳时间为当前时间

* 客户端内部同样也维护着一个定时器，每次定时器超时，判断当前是否已经有一段时间没有跟网关通信了，此时将发出心跳消息进行保活

* 当每个定时器到期时，检查客户端的心跳时间距离当前时间是否已经超过一个阈值了，如超过那么认为该客户端已经失连，将清除掉该连接

需要注意的是，客户端的定时器应该小于网关层给每个连接加上的定时器。

![keepalive](https://ivanzz1001.github.io/records/assets/img/distribute/im/keepalive.png)

## 3. 容错设计
网关有可能宕机，此时要考虑到这种情况下的容错处理。这里的原则有两条：

* 客户端一旦发现前面连接的网关宕机，将尝试重连

* 内部服务要通过网关层应答给客户端的信息，一旦发现由于网关宕机无法发出，将直接丢弃，由客户端重新尝试连接

以下来详细解释一下这两个原则。

### 3.1 客户端重连
客户端内部维护着一个发出消息的消息队列，仅在收到服务器的处理应答之后才可以从其中清除相应的消息。注意，这里每个客户端的消息ID需要做到严格递增。

![messagequeue](https://ivanzz1001.github.io/records/assets/img/distribute/im/messagequeue.png)
比如，上图中发出但是未收到应答的消息有三条，消息ID依次递增，分别是100、101、102。此时，如果收到服务器应答消息101已经被确认处理，那么在这个需要之前的消息100以及101都可以被认为已经被服务器正常接收并且处理完毕，此时可以从消息队列中删除掉需要101之前的消息了。

反之，客户端同时还维护另外一个定时器，一段时间没收到连接的网关的消息，将向网关发出心跳消息，如果仍然没有回复则认为网关出现异常，将重新走正常的登录流程尝试选择另外一台网关登录。重连之后，将重新发送消息队列中已经存在的消息。

### 3.2 重连策略
当一台网关出现问题需要客户端进行重连时，还需要考虑到不要因为重连问题导致其他网关服务器也受到影响，产生雪崩效应，此时还需要考虑以下几点：

* 打散重连时间： 需要进行重连的客户端，在一个时间范围内选择一个随机的时间，这样将这些客户端的重连时间打散，不至于一下子都连接上来

* 指数退避： 一次重连不上时，客户端还需要再尝试进行多次重连，然而重连的时间需要像TCP协议那样在阻塞恢复时做指数退避，即第一次重连时间是1秒后，第二次2秒后，第三次4秒后，等等。这个策略也是为了避免由于重连导致的服务雪崩

* 服务器保护： 上面两条是客户端的重连策略，然而服务器自身也需要进行保护，当服务器判断自己当前的负载到一定程度时，将拒绝客户端的连接请求

### 3.3 内部服务丢弃应答消息
同样的内部服务也只是通过网关层与客户端进行通信，当处理了一些消息之后需要应答客户端，此时发现对应的网关已经宕机，那么应该丢弃掉这些应答消息，等待客户端重连之后重新将前面没有收到应答的消息发出来。

如果是这个处理原则的话，对应的就需要服务器的逻辑中做到“幂等性(idempotent)”，即同一个操作，一次请求与多次请求的结果是一样的。比如，逻辑服务器可以通过客户端的消息ID来判断这条消息之前是否已经被处理过，如果是的话可以直接忽略处理，应答即可。

## 4. 服务保证
每个网关服务器可以容纳的长连接总数是固定的，到了一定程度系统资源就消耗的差不多了，应答的延迟也提高了。所以，网关层还需要考虑到服务的可用性。

比如，可以向管理网关的服务器上报如下数据：

* 当前维护的连接数量

* 当前应答延迟指标，90%的延迟到多少，99%的延迟到多少，等等

* 当前系统资源的消耗情况，比如CPU占用、内存占用等等

这样，可以有依据来判断该网关是否还能继续接收新的连接，如果不能接收连接，可以返回一批当前可用的其他网关服务列表给客户端重新发起连接，同时将当前不可用的网关从返回给客户端的网关列表中删除，这样下次就不会再来这个网关进行连接。

![qos](https://ivanzz1001.github.io/records/assets/img/distribute/im/qos.png)

如上图中，有如下步骤：

* 网关都向网关管理服务上报自己当前的服务状态，管理服务发现网关A已经接近服务极限，此时将通知网关A此时不能再接收新的连接，同时还告知当前可用的网关B和C地址

* 客户端向网关A发起请求，此时网关A拒绝该连接请求，并且返回网关B和C的服务列表给客户端

* 客户端选择网关C进行连接

可以看到，这实际上是“服务降级”的一种做法。

在某台网关服务降级之后，还可以针对具体的服务来进行优先级排列，即在当前负载的情况下，优先处理哪一类的客户端请求，而更低优先级的请求可以先不处理，比如微信在[DAGOR论文](https://mp.weixin.qq.com/s/uv4WkTIPvDCFlvKAEXrT2g)中阐述了微信内部的服务优先级：

![wechat](https://ivanzz1001.github.io/records/assets/img/distribute/im/wechat.jpg)





<br />
<br />

**[参看]:**

1. [IM服务器设计-网关接入层](https://www.codedump.info/post/20190818-im-msg-gate/)

2. [单点登录（SSO），从原理到实现](https://cloud.tencent.com/developer/article/1166255)

3. [单点登录（SSO）的设计与实现](https://ken.io/note/sso-design-implement)

4. [一套海量在线用户的移动端IM架构设计实践分享](http://www.52im.net/thread-812-1-1.html)

<br />
<br />
<br />

