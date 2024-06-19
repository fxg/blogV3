---
layout: post
title:  "Bundler::GemNotFound Could not find rake 之 Ruby On Rails Docker cron 运行问题解决办法"
date:   2019-07-17 09:00:00 +0800
categories: [技术]
tags: [Ruby On Rails, Docker, cron]
---

在 Docker 中运行 Ruby On Rails 已经是比较普遍，就像早几年使用 rvm 一样。

- Table of Contents
{:toc .large-only}

从直接安装配置各种开发环境，到与系统环境隔离（Ruby 的 rvm，Python 的 virtualenv），再到现在 Docker 化，运维们为了给自己省事，真是不遗余力，当然碰到的问题也大都相似，大概率牵扯环境变量，rvm 在 cron 运行的问题，看 Google 的搜索列表页就知道，不过好在 rvm 有条命令可以直接解决：

``` bash
rvm cron setup
```

现在迁移到 Docker 上，也碰到了在 cron 中无法执行定时任务的问题，完整的错误信息一般如下：

```
Bundler::GemNotFound: Could not find rake-12.3.2 in any of the sources
```

解决办法分如下两步：

## 1、在 Dockerfile 中增加这么一行：

``` dockerfile
RUN printenv | sed 's/^\(.*\)$/export \1/g' > /root/project_env.sh
```

主要作用就是将环境变量存在一个文件中，内容一般如下：


``` bash
export RUBY_MAJOR=2.5
export HOSTNAME=***
export RUBYGEMS_VERSION=3.0.3
export HOME=/root
export BUNDLE_APP_CONFIG=/usr/local/bundle
export RUBY_VERSION=2.5.5
export PATH=/usr/local/bundle/bin:/usr/local/bundle/gems/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
export BUNDLE_PATH=/usr/local/bundle
export GEM_HOME=/usr/local/bundle
export RUBY_DOWNLOAD_SHA256=***
export PWD=/app
export BUNDLE_SILENCE_ROOT_WARNING=1
```

## 2、在你要执行的 cron 命令中增加设置环境变量的命令：“source /root/project_env.sh”。

比如如下 cron 是会报错的，报错内容一如上面提示：

``` crontab
*/10 * * * * root /bin/bash -l -c 'cd /app && rake your:job --silent >> /app/log/cron.log 2>&1'
```

增加环境变量的初始化后，便可正常执行：

``` crontab
*/10 * * * * root /bin/bash -l -c 'source /root/project_env.sh && cd /app && rake your:job --silent >> /app/log/cron.log 2>&1'
```

## 附1、如何在 Debian 系的 Docker ROR 系统中增加 cron 任务：

首先，在你的 ROR web project 中新建目录 cron_jobs，此目录下的文件都包含一行或行调度命令，比如其中一个文件叫：myjob，内容如上，然后在 Dockerfile 中增加如下命令行：

``` dockerfile
COPY ./cron_jobs/* /etc/cron.d/
RUN chmod 644 /etc/cron.d/*
RUN crontab /etc/cron.d/*
```

其次，就是别忘了启动你的 Dokcer container 中的 cron 进程。

## 附2、列出最简配置文件：

Dockerfile：

``` dockerfile
FROM ruby:2.5
RUN apt-get update -qq && apt-get install -y nodejs postgresql-client cron

WORKDIR /app

COPY Gemfile /app/Gemfile
COPY Gemfile.lock /app/Gemfile.lock
RUN bundle install

COPY ./cron_jobs/* /etc/cron.d/
RUN chmod 644 /etc/cron.d/*
RUN crontab /etc/cron.d/*
RUN printenv | sed 's/^\(.*\)$/export \1/g' > /root/project_env.sh

# Add a script to be executed every time the container starts.
COPY entrypoint.sh /usr/bin/
RUN chmod +x /usr/bin/entrypoint.sh
ENTRYPOINT ["entrypoint.sh"]
EXPOSE 3000

# Start the main process.
CMD ["rails", "server", "-b", "0.0.0.0"]
```

entrypoint.sh：

``` bash
#!/bin/bash
set -e

# Remove a potentially pre-existing server.pid for Rails.
rm -f /app/tmp/pids/server.pid
rm -f /app/tmp/pids/sidekiq.pid

# Then exec the container's main process (what's set as CMD in the Dockerfile).
exec "$@"
```

docker-compose.yml：

``` yaml
version: '3'

services:
  web:
    depends_on:
      - 'postgres'
      - 'redis'
    build: ./web
    restart: always
    volumes:
      - './web:/app'

  sidekiq:
    depends_on:
      - 'web'
      - 'gearman-worker'
    build: ./web
    restart: always
    command: bash -c "cron && sidekiq -C config/sidekiq.yml"
    volumes:
      - './web:/app'

  postgres:
    image: 'postgres:10'
    container_name: postgresql
    restart: always
    volumes:
      - 'postgres:/var/lib/postgresql/data'

  redis:
    image: 'redis:5'
    container_name: redis
    restart: always
    volumes:
      - 'redis:/data'
volumes:
  redis:
  postgres:
```
