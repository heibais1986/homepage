---
publishDate: 2025-05-14T00:00:00Z
title: how to setup a n8n server
image: https://i2.hdslb.com/bfs/archive/1b88ce7e29010144d02464e595553e2860316eb4.jpg
tags:
  - n8n
  - devops
---
n8n在GitHub已经有75K 超高Star，今天我们就来使用中文快速搭建。

GitHub地址：https://github.com/n8n-io/n8n

下载n8n汉化包 

首先我们会用到这个开源项目（n8n的汉化包）https://github.com/other-blowsnow/n8n-i18n-chinese/releases进去之后，在Release页面，下载和自己n8n版本对应的editor-ui.tar.gz文件。

比如我的n8n是1.90.2版本，我就下载了1.90.2版本的汉化包

editor-ui.tar.gz下载下来之后，解压就得到一个名叫dist的文件夹，我们获取这个文件夹所在的路径，在原来的n8n docker-compose.yml文件里面添加两行配置（如下图红框所示）
![n8n-1](/images/n8n-1.jpg)


```version: '3.3'
services:
n8n:
image: n8nio/n8n:1.90.2
container_name: n8n
restart: always
ports:
- "29764:29764"
volumes:
- n8n_data:/root/.n8n
# 卷映射：将宿主机的汉化文件目录映射到容器内的UI目录
- /root/n8n/dist:/usr/local/lib/node_modules/n8n/node_modules/n8n-editor-ui/dist
environment:
- NODE_ENV=production
- N8N_DEFAULT_LOCALE=zh-CN # 设置默认语言为简体中文
- N8N_SECURE_COOKIE=false
- N8N_HOST=n8n.docman.dpdns.org
- N8N_PORT=29764
# 可以根据需要添加其他环境变量
volumes:
n8n_data:
external: true
```

然后使用docker-compose up -d命令即可启动。

这里面有个坑，如果你想使用X等 需要配置重定向URL的节点

![n8n-2](/images/n8n-2.png)

那么启动前需要配置N8N_HOST=外网IP，如果不指定默认是localhost，到时候X节点是无法授权成功的。

部署成功之后直接访问 http://localhost:29764，如果是部署到云服务器，请访问 http://外网IP:29764

注意：最好下载跟自己n8n版本号一致的汉化包。每次升级n8n也最好去下载对应版本的汉化包，解压后替换原汉化包。
