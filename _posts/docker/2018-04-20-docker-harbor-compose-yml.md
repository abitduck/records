---
layout: post
title: Harbor docker-compose文件分析
tags:
- docker
categories: docker
description: Harbor docker-compose文件分析
---


本节我们主要分析一下Harbor中docker-compose.yml文件，以了解Harbor整个系统的一个整体架构及工作状况。



<!-- more -->

## 1. docker-compose.yml文件
{% highlight string %}
version: '2'
services:
  log:
    image: vmware/harbor-log:v1.4.0
    container_name: harbor-log 
    restart: always
    volumes:
      - /var/log/harbor/:/var/log/docker/:z
      - ./common/config/log/:/etc/logrotate.d/:z
    ports:
      - 127.0.0.1:1514:10514
    networks:
      - harbor
  registry:
    image: vmware/registry-photon:v2.6.2-v1.4.0
    container_name: registry
    restart: always
    volumes:
      - /data/registry:/storage:z
      - ./common/config/registry/:/etc/registry/:z
    networks:
      - harbor
    environment:
      - GODEBUG=netdns=cgo
    command:
      ["serve", "/etc/registry/config.yml"]
    depends_on:
      - log
    logging:
      driver: "syslog"
      options:  
        syslog-address: "tcp://127.0.0.1:1514"
        tag: "registry"
  mysql:
    image: vmware/harbor-db:v1.4.0
    container_name: harbor-db
    restart: always
    volumes:
      - /data/database:/var/lib/mysql:z
    networks:
      - harbor
    env_file:
      - ./common/config/db/env
    depends_on:
      - log
    logging:
      driver: "syslog"
      options:  
        syslog-address: "tcp://127.0.0.1:1514"
        tag: "mysql"
  adminserver:
    image: vmware/harbor-adminserver:v1.4.0
    container_name: harbor-adminserver
    env_file:
      - ./common/config/adminserver/env
    restart: always
    volumes:
      - /data/config/:/etc/adminserver/config/:z
      - /data/secretkey:/etc/adminserver/key:z
      - /data/:/data/:z
    networks:
      - harbor
    depends_on:
      - log
    logging:
      driver: "syslog"
      options:  
        syslog-address: "tcp://127.0.0.1:1514"
        tag: "adminserver"
  ui:
    image: vmware/harbor-ui:v1.4.0
    container_name: harbor-ui
    env_file:
      - ./common/config/ui/env
    restart: always
    volumes:
      - ./common/config/ui/app.conf:/etc/ui/app.conf:z
      - ./common/config/ui/private_key.pem:/etc/ui/private_key.pem:z
      - ./common/config/ui/certificates/:/etc/ui/certificates/:z
      - /data/secretkey:/etc/ui/key:z
      - /data/ca_download/:/etc/ui/ca/:z
      - /data/psc/:/etc/ui/token/:z
    networks:
      - harbor
    depends_on:
      - log
      - adminserver
      - registry
    logging:
      driver: "syslog"
      options:  
        syslog-address: "tcp://127.0.0.1:1514"
        tag: "ui"
  jobservice:
    image: vmware/harbor-jobservice:v1.4.0
    container_name: harbor-jobservice
    env_file:
      - ./common/config/jobservice/env
    restart: always
    volumes:
      - /data/job_logs:/var/log/jobs:z
      - ./common/config/jobservice/app.conf:/etc/jobservice/app.conf:z
      - /data/secretkey:/etc/jobservice/key:z
    networks:
      - harbor
    depends_on:
      - ui
      - adminserver
    logging:
      driver: "syslog"
      options:  
        syslog-address: "tcp://127.0.0.1:1514"
        tag: "jobservice"
  proxy:
    image: vmware/nginx-photon:v1.4.0
    container_name: nginx
    restart: always
    volumes:
      - ./common/config/nginx:/etc/nginx:z
    networks:
      - harbor
    ports:
      - 80:80
      - 443:443
      - 4443:4443
    depends_on:
      - mysql
      - registry
      - ui
      - log
    logging:
      driver: "syslog"
      options:  
        syslog-address: "tcp://127.0.0.1:1514"
        tag: "proxy"
networks:
  harbor:
    external: false
{% endhighlight %}

这里我们再给出Harbor的整个架构图：

![docker-harbor-arch](https://ivanzz1001.github.io/records/assets/img/docker/docker_harbor_arch.png)







<br />
<br />

**[参考]**

1. [harbor官网](https://github.com/vmware/harbor)

2. [vmware harbor](http://vmware.github.io/harbor/)

3. [habor documents](https://github.com/vmware/harbor/tree/master/docs)

<br />
<br />
<br />
