---
layout: post
title:  "使用 acme.sh 生成 Let’s Encrypt 泛域名证书"
date:   2019-03-06 15:39:10 +0800
categories: [技术]
tags: [acme.sh, Let’s Encrypt, 泛域名证书]
---

- Table of Contents
{:toc .large-only}

系统环境：Debian 9 and Nginx。

## 1、安装 ##

``` bash
curl  https://get.acme.sh | sh
```

会自动安装到当前用户目录主目录下的 .acme.sh 目录下。

别人一般会再设置 alisa，我一般都直接进入其目录，这看个人喜好。

## 2、配置 DNS 环境变量 ##

在获取证书前，需要先确认你的 DNS 解析商，以 DNSPod 为例，你需要先设置相关的环境变量：

``` bash
export DP_Id="your-api-token-id"
export DP_Key="your-api-token"
```

至于对应参数的值如何获取，可登录：https://www.dnspod.cn/console/user/security , 增加专用 API token，以替换上述对应值。

更多 DNS 解析商的设置，请移步：[这里][more-acme-sh-dnsapi]

## 3、获取证书 ##

进入执行目录：~/.acme.sh，执行：

``` bash
./acme.sh --issue -d "erma.eu.org" -d "*.erma.eu.org" --server https://acme-v02.api.letsencrypt.org/directory --dns dns_dp
```

## 4、设置 Nginx 配置文件 ##

先建立证书存放目录，比如：/path/to/your/keys/。

在你的 Nginx 配置文件对应的 server 中，增加：

``` conf
listen 443 ssl http2;
listen [::]:443 ssl http2;
server_name you-domain-name.com;

ssl on;
ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
ssl_certificate "/path/to/your/keys/erma.eu.org.crt";
ssl_certificate_key "/path/to/your/keys/erma.eu.org.key";
```


## 5、安装证书到预定证书存放目录 ##

``` bash
./acme.sh --installcert -d "erma.eu.org" -d "*.erma.eu.org" --key-file /path/to/your/keys/erma.eu.org.key --fullchain-file /path/to/your/keys/erma.eu.org.crt --reloadcmd "sudo service nginx force-reload"
```

一般情况下，“sudo service nginx force-reload”，会提示输入密码，如果是放在 crontab 执行的话，会造成重启 Nginx 失败，所以，有必要将 Nginx 命令放入 sudo 免密码执行中。

在 visudo 对应的用户，比如 www-data ，增加如下行：

```
www-data        ALL=(ALL) NOPASSWD: /usr/sbin/service nginx force-reload
```

## 6、收尾 ##

设置自动升级：

``` bash
./acme.sh --upgrade --auto-upgrade
```

设置周期更新证书：

``` bash
./acme.sh --cron
```

在生成 Let’s Encrypt 泛域名证书的时候，acme.sh 比 certbot 好在什么地方，我不知道，我就知道 acme.sh 方便。

[more-acme-sh-dnsapi]: https://github.com/Neilpang/acme.sh/blob/master/dnsapi/README.md
