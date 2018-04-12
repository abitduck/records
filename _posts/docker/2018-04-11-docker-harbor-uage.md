---
layout: post
title: Harbor使用手册
tags:
- docker
categories: docker
description: Harbor使用手册
---


本章将会介绍一下Harbor的基本使用， 主要包括以下方面的内容：

* Harbor中工程(projects)的管理

* project中成员(members)的管理

* 复制工程到远程仓库(remote registry)

* 查询工程(projects)及repositories

* 系统管理员身份对Harbor系统的管理

* 使用docker client拉取及推送镜像

* 为repositories添加描述

* 删除repositories及镜像

* Content Trust


<!-- more -->

## 1. 基于角色的访问控制

Harbor提供了基于角色的访问控制（Role Based Access Control, RBAC):

![docker-harbor-rbac](https://ivanzz1001.github.io/records/assets/img/docker/docker_harbor_rbac.png)


Harbor通过工程（projects)的方式来管理镜像(images)。在添加用户到工程中时，可以是如下三种角色中的一种：

* ```Guest```: 指定工程（project)中的Guest用户只有只读访问权限

* ```Developer```: 工程中的Developer用户具有读写访问权限

* ```ProjectAdmin```: 当创建一个新的工程时，你就会被默认指定为该工程的```ProjectAdmin```角色。除了具有读写权限之外，```ProjectAdmin```还有一些管理特权，例如向该工程添加/移除成员、启动一个vulnerability扫描

另外除了这上面这三种角色，还有两个系统层面的角色：

* ```SysAdmin```: SysAdmin角色的用户具有最高管理权限。除了上面提到的一些权限之外，```SysAdmin```用户可以列出所有的工程(projects)； 将一个普通用户设置为管理员； 删除用户； 针对所有镜像设置vulnerability扫描策略。公有工程```library```也是由系统管理员所拥有。

* ```Anonymous```: 当一个用户并没有登录的时候，该用户会被认为是一个匿名用户。一个匿名用户并没有访问私有工程(private project)的权限, 而对于公有工程(public project)拥有只读访问权限。


视频示例: [Tencent Video](https://v.qq.com/x/page/l0553yw19ek.html)


## 2. 用户账户(User Account)
Harbor支持两种身份验证方式：

* **Database(db_auth)**： 这种情况下，所有用户被存放在一个本地数据库

在这种模式下，一个用户可以在Harbor系统中进行注册。如果要禁止```自行注册```功能，在初始化配置时请参看[installation guide](https://github.com/vmware/harbor/blob/master/docs/installation_guide.md)； 或者是在```Administraor Options```中禁止该特性。当self-registration被禁止后，系统管理员可以添加用户到Harbor中。

当注册或添加一个新的用户到Harbor中时，Harbor系统中的用户名、email必须是唯一的。密码至少要有8个字符长度，并且至少要包含一个大写字母(uppercase letter)、一个小写字母(lowercase letter)以及一个数字(numeric character)

当你忘记密码的时候，你可以按如下的步骤来重设密码:
<pre>
1) 在注册页面点击"Forgot Password" 链接

2) 输入你注册时所用的邮箱地址， 然后Harbor系统会发送一封邮件给你来进行密码重设

3) 在收到邮件之后，单击邮件中给出的链接地址，然后会跳转到一个密码重设的Web页面

4) 输入新的密码并单击"Save"按钮
</pre>

* **LDAP/Active Directory(ldap_auth)**: 在这种认证模式下，用户的credentials都被存放在外部的LDAP或AD服务器中，用户在那边完成认证后可以直接登录到Harbor系统。

当一个LDAP/AD用户通过```username```和```password```的方式登录进系统时，Harbor会用```LDAP Search DN```及```LDAP Search Password```绑定LDAP/AD服务器（请参看[installation guide](https://github.com/vmware/harbor/blob/master/docs/installation_guide.md))。假如成功的话，Harbor会在LDAP的```LDAP Base DN```目录及其子目录来查询该用户。通过```LDAP UID```指定的一些属性(比如: uid、cn)会与```username```一起共同来匹配一个用户。假如被成功匹配，用户的密码会通过一个发送到LDAP/AD服务器的bind request所验证。假如LDAP/AD服务器使用自签名证书(self-signed certificate)或者不受信任的证书的话，请不要检查```LDAP Verify Cert```。

在LDAP/AD认证模式下，不支持```self-registration```、删除用户、修改密码、重设密码等功能，这是因为用户是由LDAP/AD系统所管理.


## 3. Harbor工程管理

Harbor中的一个工程包含了一个应用程序所需要的所有repositories。在工程创建之前，并不允许推送镜像到Harbor中。Harbor对于project采用基于角色的访问控制。在Harbor中projects有两种类型：

* ```Public```: 所有的用户都具有读取public project的权限， 其主要是为了方便你分享一些repositories

* ```Private```: 一个私有project只能被特定用户以适当的权限进行访问


在登录之后，你就可以创建一个工程(project)。默认情况下，创建的工程都是私有的，你可以通过在创建时选中```Access Level```复选框来使创建的工程变为public的：

![harbor-create-project](https://ivanzz1001.github.io/records/assets/img/docker/harbor_create_project.png)


在工程被创建完成之后，你可以通过导航标签浏览```repositories```，```members```, ```logs```,```replication```以及```configuration```:

![harbor-browse-project](https://ivanzz1001.github.io/records/assets/img/docker/harbor_browse_project.png)

你可以通过单击```Logs```选项卡来查看所有的日志。你也可以使用```username```, ```operations```以及```Advanced Search```中的日期来过滤日志信息：

![log-search-advanced](https://ivanzz1001.github.io/records/assets/img/docker/log_search_advanced.png)

可以通过单击```Configuration```选项卡设置工程相关属性：

* 选中```Public```复选框，你就可以将该工程下的所有repositories设置为公有访问权限

* 通过使能```Enable content trust```复选框来阻止从工程拉取未被登记的镜像

* 通过选择```Prevent vulnerable images from running```复选框和改变vulnerabilities的安全级别，以阻止vulnerable镜像被拉取。假如一个镜像的安全级别高于大于等于当前所设置的级别的话，Image将不能够被拉取。

* 为了激活对push到工程中的新镜像进行vulnerability扫描，可以选中```Automatically scan images```复选框

![harbor-project-config](https://ivanzz1001.github.io/records/assets/img/docker/harbor_project_config.png)


## 4. 管理工程中的用户成员

**1） 添加成员**

你可以向一个已存在的工程添加项目成员，并为其指定相应的角色。在LDAP/AD认证模式下，你也可以添加一个LDAP/AD用户使其成为工程中的一员：

![new-add-member](https://ivanzz1001.github.io/records/assets/img/docker/new_add_member.png)

**2) 更新及移除成员**

可以批量选中工程中的成员，然后修改他们的角色； 也可以删除某些成员：

![harbor-update-member](https://ivanzz1001.github.io/records/assets/img/docker/harbor_update_member.png)

## 5. 复制镜像

镜像复制被用于从一个Harbor实例向另一个Harbor实例复制repositories。

该功能是面向工程的(project-oriented), 而一旦系统管理员为该工程设置了相应的规则，当[触发条件](https://github.com/vmware/harbor/blob/master/docs/user_guide.md#trigger-mode)被触发的话该工程下所有匹配相应规则的repositories都会被复制到远程仓库中。
每一个repository都会启动一个```job```来进行复制。假如该工程在远程镜像仓库中并不存在，则会自动的创建出一个新的工程。但是假如该工程已经存在并且被用户配置为没有写权限的话，则这个同步过程会失败。这里注意：```项目成员信息将不会被复制```

根据不同的网络状况，这个复制过程可能会有一定的延迟。假若是因为网络的原因导致镜像同步失败，则在几分钟之后Harbor会尝试再进行同步（该同步过程会一直进行，直到网络恢复，同步成功）

注意： Harbor 0.35版本之前和之后的不同实例不能进行相互同步。


### 5.1 创建复制规则
可以通过创建规则来配置复制策略。





<br />
<br />

**[参看]**

1. [Harbor User Guide](https://github.com/vmware/harbor/blob/master/docs/user_guide.md)


<br />
<br />
<br />
