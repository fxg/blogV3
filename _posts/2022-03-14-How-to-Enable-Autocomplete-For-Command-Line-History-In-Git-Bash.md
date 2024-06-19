---
layout: post
title: "如何在 Windows 版 Git Bash 中启用上下键自动补全历史命令"
date: 2022-03-14 15:50:00 +0800
categories: [技术]
tags: [Git Bash, Fzf]
---

本文主要参考自[这篇文章](https://www.usessionbuddy.com/post/How-to-Enable-Autocomplete-For-Command-Line-History-In-Bash-and-Zsh/){:target="_blank"}。

- Table of Contents
{:toc .large-only}

## 1、下载 Fzf

首先在 Git bash 环境根目录下：

``` shell
git clone --depth 1 https://github.com/junegunn/fzf.git ~/.fzf
```

## 2、安装

``` shell
~/.fzf/install 
```

基本默认都是：y 就好了：

```
Downloading bin/fzf ...
Already exists
Checking fzf executable ... 0.19.0 
Do you want to enable fuzzy auto-completion? ([y]/n) y 
Do you want to enable key bindings? ([y]/n) y
Generate /root/.fzf.bash ... OK
Do you want to update your shell configuration files? ([y]/n) y
Update /root/.bashrc:
[ -f ~/.fzf.bash ] && source ~/.fzf.bash
Added
Finished. Restart your shell or reload config file. source ~/.bashrc 
```

## 3、启用 Fzf

这时可以重新打开 Git bash，也可以执行如下命令，来启用 Fzf：

``` shell
source ~/.bashrc
```

## 4、上下键自动补全历史命令

修改：.fzf/shell/key-bindings.bash，在末尾增加如下键绑定：

``` shell
bind '"\e[A": history-search-backward'
bind '"\e[B": history-search-forward'
bind '"\eOA": history-search-backward'
bind '"\eOB": history-search-forward'
```

重新启用一下 Fzf 后，大功告成！