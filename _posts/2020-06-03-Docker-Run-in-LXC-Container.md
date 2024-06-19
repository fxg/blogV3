---
layout: post
title: "在 LXC 容器中运行 Docker image"
date: 2020-06-03 17:10:00 +0800
categories: [技术]
tags: [LXC, Docker]
---

我还是觉得有必要记下这个问题的解决办法：

LXC 容器内运行 Docker image，出现错误：“starting container process caused "process_linux.go:449: container init caused "join session keyring: create session key: disk quota exceeded"”。

在文件：/etc/modules-load.d/modules.conf 中增加如下：

```
aufs
overlay
```

重启一下 LXC 容器，可以了！