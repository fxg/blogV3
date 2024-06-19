---
layout: post
title: "使用 logrotate 管理 Docker 容器的日志"
date: 2019-11-28 11:05:00 +0800
categories: [技术]
tags: [logrotate, Docker]
---

> 其实 Docker 使用 logrotate 管理日志，纯属多此一举，Docker 本身就具备切割日志文件功能。
{: .prompt-info }

文件 /etc/docker/daemon.json：
``` json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3",
    "compress": "true"
  }
}
```

之前原文：

Docker Container 的 log 如不管理的话，很容易把小磁盘的空间写满，所以，迫切用传统办法将日志管理起来。

编辑 /etc/docker/daemon.json：

``` json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
```

然后重启 Dockers 服务，比如我是 Debian 环境：

``` shell
sudo service docker restart
```

安装 logrotate 后，确认配置目录是否是常规的，比如 /etc/logrotate.conf 中包括 include /etc/logrotate.d 配置，那么新建文件 /etc/logrotate.d/docker：

``` conf
/var/lib/docker/containers/*/*.log {
  rotate 7
  daily
  compress
  missingok
  delaycompress
  copytruncate
}
```

Done!