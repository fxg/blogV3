---
layout: post
title: "使用 Github Action 构建并推送 Docker 镜像到 Gitlab 的镜像托管中心"
date: 2020-11-17 15:57:00 +0800
categories: [技术]
tags: [Github Action, Docker, Gitlab]
---

Github 和 Gitlab 两家都有免费的持续集成和发布功能，本文综合 Github 和 Gitlab 两者免费账户的各自特点，取长补短，以达到一定程度的优化体验。

- Table of Contents
{:toc .large-only}

## 起因

Github VS Gitlab 对比如下：

|功能|Github|Gitlab|
|-|-|
|私人仓库数|无限制|无限制|
|Actions|2000 分钟，或自建 runner|2000 分钟，或自建 runner|
|Actions 和 Packages 存储空间|500MB|10GB|
|Packages 上行流量|1GB|暂未查到|

Github 500MB 的镜像存储空间相对有些掣肘。

两家都支持用户自建 runner，我之前都是自建服务器来构建 Docker 镜像，但是性价比不高，虽然两家都提供免费 2000 分钟的 CI/CD 时间，但是因为 Github 会启动一个双核 CPU，7GB 内存的虚拟机来跑，而 Gitlab 是共享十几台服务器来跑，所以 Github 可以说相当良心了。同一个镜像的构建，Github 的虚拟机跑不到8分钟，而我 1 核 1G 内存的高性能 VPS，则要跑17分钟左右，对比很明显了。

我们的目标，版本库同时使用 Github 和 Gitlab（Gitlab 只做 Push，以作备份），源码推送到 Github 后，开始启动 Action，待镜像构建完毕即推送到 Gitlab Container Registry，然后我们从 Gitlab 下载镜像来进行后续的应用部署。

## 新建或者旧有 Github 仓库中增加推送到 Gitlab 的链接

``` shell
git remote set-url --add --push origin your-gitlab-repostory-link
```

用命令：

``` shell
git remote -v
```

查看最后的链接，大约是这个样子：

``` log
origin  git@github.com:your-user/your-repo.git (fetch)
origin  git@gitlab.com:your-user/your-repo.git (push)
origin  git@github.com:your-user/your-repo.git (push)
```

## 申请 Gitlab token

这一步是为了让 Github 能使用 Gitlab 相关的功能，在 Gitlab 网站，如下路径：User Settings > Access Tokens，新建 Token，只勾择如下功能即可：read_registry, write_registry。或许也可以取消 read_registry 功能，但我没试。

生成完毕后，复制出现的 Token，备用。


## 设置 Github 中的仓库

进入仓库的 Settings > Secrets 中，点击 New repostory secrets 按钮，增加名为：GITLAB_TOKEN 的项，值就是上一步申请的 Token。
然后再增加 GITLAB_USERNAME 项，值就是你的 Gitlab 登录名。

## 创建 Github Action 文件

按照 Github 的规则，在仓库根目录下创建如下两级目录：.github\workflows。在新创建的目录下，创建 YAML 文件，比如叫：build-push-docker-image.yml。内容如下：

``` yaml
name: build and push docker image to github package

on:
  push:
    branches: master

jobs:
  path-context:
    runs-on: ubuntu-latest
    steps:
      -
        name: Checkout
        uses: actions/checkout@v2
      -
        name: Set up QEMU
        uses: docker/setup-qemu-action@v1
      -
        name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v1
      -
        name: Login to DockerHub
        uses: docker/login-action@v1 
        with:
          username: {% raw %}${{ secrets.DOCKERHUB_USERNAME }}{% endraw %}
          password: {% raw %}${{ secrets.DOCKERHUB_TOKEN }}{% endraw %}
      -
        name: Login to Gitlab Container Registry
        uses: docker/login-action@v1 
        with:
          registry: registry.gitlab.com
          username: {% raw %}${{ secrets.GITLAB_USERNAME }}{% endraw %}
          password: {% raw %}${{ secrets.GITLAB_TOKEN }}{% endraw %}
      -
        name: Build and push
        uses: docker/build-push-action@v2
        with:
          context: .
          file: ./Dockerfile
          push: true
          tags: registry.gitlab.com/{% raw %}${{ secrets.GITLAB_USERNAME }}{% endraw %}/your-repo/your-app:latest
          build-args: |
            arg1=var1
            arg2=var2
      -
        name: Image digest
        run: echo {% raw %}${{ steps.docker_build.outputs.digest }}{% endraw %}
```

上述内容主要来自于：[Docker 官方 build-push-action 库的描述]，我只是增加了对 Gitlab Container Registry 推送描述。

如上设置完毕后，实现了两库推送（Github，Gitlab），使用 Github Action，完成镜像构建并推送到 Gitlab。

[Docker 官方 build-push-action 库的描述]:https://github.com/docker/build-push-action