---
layout: post
title:  "Docker Build 阶段的代理设置"
date:   2018-11-29 09:45:10 +0800
categories: [技术]
tags: [Docker]
---

Docker 在构建阶段，如果要通过代理下载文件的话，需要使用 arg 命令，如下所示 Dockerfile ：

``` dockerfile
FROM ruby as ruby

ARG https_proxy=https://192.168.1.2:8080
ARG http_proxy=http://192.168.1.2:8080

WORKDIR /tmp

RUN apt update \
  && apt -y -q upgrade \
  && apt install -q -y --no-install-recommends --no-install-suggests \
    ruby-dev unzip wget tar openssl xvfb chromium lsof \
    chrpath libxft-dev libfreetype6 libfreetype6-dev libfontconfig1 libfontconfig1-dev \
  && wget https://chromedriver.storage.googleapis.com/2.44/chromedriver_linux64.zip \
  && unzip -o chromedriver_linux64.zip -d /usr/local/bin \
  && rm -f chromedriver_linux64.zip \
  && wget https://bitbucket.org/ariya/phantomjs/downloads/phantomjs-2.1.1-linux-x86_64.tar.bz2 \
  && tar -xvjf phantomjs-2.1.1-linux-x86_64.tar.bz2 \
  && mv phantomjs-2.1.1-linux-x86_64 /usr/local/lib \
  && ln -s /usr/local/lib/phantomjs-2.1.1-linux-x86_64/bin/phantomjs /usr/local/bin \
  && rm -f phantomjs-2.1.1-linux-x86_64.tar.bz2 \
  && rm -rf /var/lib/apt/lists/*

COPY Gemfile ./

RUN gem install bundler \
  && bundle install
```

https_proxy, http_proxy 更改为实际的代理地址。
