---
layout: post
title:  "基于 alpine 的 Docker 容器，cron 无法执行定时脚本"
date:   2018-12-07 16:39:10 +0800
categories: [技术]
tags: [Docker, cron, alpine]
---

> 2019-07-30 更新附加完整 Dockerfile
{: .prompt-info }

Alpine 内嵌的是 BusyBox，使用 BusyBox 的 crond 服务，内含多个默认目录：

``` crontab
# min   hour    day     month   weekday command
*/15    *       *       *       *       run-parts /etc/periodic/15min
0       *       *       *       *       run-parts /etc/periodic/hourly
0       2       *       *       *       run-parts /etc/periodic/daily
0       3       *       *       6       run-parts /etc/periodic/weekly
0       5       1       *       *       run-parts /etc/periodic/monthly
```

将需要执行的脚本放入对应目录即可完成定时调度任务。

 放入目录的脚本也有要求：

 - 1、可执行
 - 2、不能带扩展名
 - 3、第一行代码类似是：#!/bin/sh

但是，如果是在 Windows 编辑脚本文件，容器运行相关脚本后可能会报错：

```
run-parts: can't execute **** No such file or directory
```

罪魁祸首可能就是因为文件格式是 dos 格式，用 Linux 上的命令 dos2unix 转一下即可。

附：基于 Python 的可连接 Mysql 数据库的 Dockerfile:

``` dockerfile
FROM python:3.7-alpine

WORKDIR /code

RUN apk add --no-cache mariadb-connector-c-dev tzdata && \
    apk add --no-cache --virtual .build-deps \
        build-base \
        mariadb-dev && \
    pip install mysqlclient && \
    apk del .build-deps

RUN cp -f  /usr/share/zoneinfo/Asia/Shanghai /etc/localtime

COPY crontabs .
RUN crontab crontabs

CMD /usr/sbin/crond -l 2 -f
```