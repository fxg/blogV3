---
layout: post
title:  "Jekyll 的安装和配置"
date:   2018-07-26 16:27:10 +0800
categories: [技术]
tags: [Jekyll]
---

> 实际的部署方式很多，现在已经改成了本地生成 _site 下的文件，通过 rsync 同步到远端网站目录下的方法。
{: .prompt-info }

Jekyll 很诱人，但是安装和维护不符合我的老旧思维，所以直到本篇文章之前都没有真正实际使用起来。

本篇先抛开 Jekyll 的各种语法，也抛开安装步骤，直接开始搞自动发布文章和配置主题！先得好看才能开始写文章，能写文章才会去了解那些语法。

- Table of Contents
{:toc .large-only}

## 1、先决条件 ##

我在 Windows 10 上编辑文章，发布到一台 Debian 9 的主机上，Windows 的本地测试环境是 Bash On Windows，部署 RVM 很顺畅。

## 2、自动发布 ##

在 Debian 的服务器上，生成一个版本库：
```bash
git init --bare blog.erma.eu.org.git
```
然后在 blog.erma.eu.org.git/hooks 目录下，增加 post-receive 文件：
```bash
#!/bin/bash -l
GIT_REPO=$HOME/git-repos/blog.erma.eu.org.git
GIT_REPO_LOCAL=$HOME/sites/data/blog.erma.eu.org
PUBLIC_WWW=$HOME/sites/www/blog.erma.eu.org

unset GIT_DIR && cd $GIT_REPO_LOCAL && git pull && jekyll build -s $GIT_REPO_LOCAL -d $PUBLIC_WWW --incremental
exit
```

在 Windows 本地，Jekyll 站点目录下：
```bash
git init
git remote add origin user@blog.erma.eu.org:~/git-repos/blog.erma.eu.org.git
git commit -a -m "init."
git push --set-upstream origin master
```

以后每次提交本地新版本文章内容，在 git push 之后，服务器端都会自动 git pull, 并增量更新站点内容。

## 3、主题配置 ##

我选择的主题是 [hydeout][jekyll-theme-hydeout]，安装步骤是 Jekyll 主题的通用安装步骤，不过配置需要稍微注意，按照文档所说，因为加了分页，所以把 index.md 改名成 index.html，并更改为一下内容：

```yaml
---
layout: index
title: Home
---
```

## 4、其他配置 ##

更改 _config.yml 文件，url，title，email，description 等更改为实际内容。

其中 theme 对应值：jekyll-theme-hydeout

另外增加：excerpt_separator: \<!--more--\>

表明摘要的结束点，这样在文章列表中就不显示全文了。

[jekyll-theme-hydeout]: https://github.com/fongandrew/hydeout
