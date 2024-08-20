---
layout: post
title: "受限环境（虚拟主机无 root 权限）下的 Dancer2 开发"
date: 2024-08-20 16:06:18 +0800
categories: [技术]
tags: [Dancer2, Perl]
---

- Table of Contents
{:toc .large-only}

## cpanm 安装

``` shell
wget -O- http://cpanmin.us | perl - -l ~/perl5 App::cpanminus local::lib
eval `perl -I ~/perl5/lib/perl5 -Mlocal::lib`
echo 'eval `perl -I ~/perl5/lib/perl5 -Mlocal::lib`' >> ~/.bashrc
echo 'export MANPATH=$HOME/perl5/man:$MANPATH' >> ~/.bashrc
```

## Perl 模块安装

``` shell
cpanm Dancer2 JSON
```

## 新建应用

当前目录假设为：/home/testorg/apps/

``` shell
dancer2 gen -a MyWebApp
```

## 修改应用配置：

修改：config.yml

增加如下：

```yaml
environment: "production"
session: "YAML"
serializer: "JSON"
```

## 修改应用代码

进入：MyWebApp 目录，修改：lib/MyWebApp.pm 文件：

``` perl
package MyWebApp;
use Dancer2;

our $VERSION = '0.1';

get '/' => sub {
    return { message => "Pong!" };
};

get '/test' => sub {
    return { message => "This is the test route!" };
};

true;
```

## cgi-bin 目录下的配置

新建，如果没有这个文件：.htaccess，内容：

```
Options +ExecCGI
AddHandler cgi-script cgi
DirectoryIndex index.cgi

RewriteEngine On

RewriteCond %{REQUEST_FILENAME} !-f
RewriteCond %{REQUEST_FILENAME} !-d
RewriteRule ^(.*)$ index.cgi/$1 [L]
```

## 调用 Dancer2 App 的通用模板

比如文件名为：index.cgi：

``` perl
#!/usr/bin/env perl
use strict;
use warnings;

use lib '/home/testorg/perl5/lib/perl5';

use Plack::Handler::CGI;

# 导入 Dancer2 应用程序
use lib "/home/testorg/apps/MyWebApp/lib";
use MyWebApp;

# 创建 CGI 适配器
my $app = MyWebApp->to_app;

Plack::Handler::CGI->new->run($app);
```

## 结尾

此时调用：https://your-domain-url/test，会显示：

``` json
{
"message": "This is the test route!"
}
```

## 备注

如果是纯 API 应用，可以删除 Dancer2 自动生成的 public、views 目录。

## 卸载自安装的 Perl 模块

删目录：

``` shell
rm -fr ~/perl5
rm -fr .cpanm
```

删环境，.bashrc 中，删除这两行：

``` bashrc
eval `perl -I ~/perl5/lib/perl5 -Mlocal::lib`
export MANPATH=$HOME/perl5/man:$MANPATH
```