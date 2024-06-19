---
layout: post
title: "在 Docker 容器中启用 IPv6"
date: 2020-06-04 10:32:00 +0800
categories: [技术]
tags: [Docker, IPv6]
---

网络上的教程很多，但是难点（与我而言）在于转发 IPv6 的数据，流水账如下：

- Table of Contents
{:toc .large-only}

## 1、确定母鸡支持 IPv6

``` shell
ping6 google.com -n -c 4
```

## 2、Docker 配置

修改文件：/etc/docker/daemon.json，增加 Docker 配置项：

``` json
{
  "ipv6": true,
  "fixed-cidr-v6": "2001:db8:1::/64"
}
```

## 3、重启 Docker

``` shell
sudo systemctl restart docker
```

## 4、配置 IPv6 NAT

``` shell
sudo ip6tables -t nat -A POSTROUTING -s 2001:db8:1::/64 ! -o docker0 -j MASQUERADE
```

## 5、测试是否已经支持 IPv6 网络

``` shell
sudo docker run --rm -t busybox ping6 -c 4 google.com
```

## 6、docker-compose 相关

如果你使用 docker-compose 作为容器启动服务，那还需要修改对应的 YAML 文件，增加 network IP 相关的信息，完整 docker-compose.yml 文件如下：

``` yaml
version: '3.7'
services:
  caddy:
    image: caddy
    container_name: caddy
    restart: always
    ports:
      - "443:443"
    volumes:
      - ./config:/etc/caddy
    networks:
      default_network:
        ipv4_address: 172.16.238.10
        ipv6_address: 2001:1234:5678::10

networks:
  default_network:
    enable_ipv6: true
    driver: bridge
    driver_opts:
      com.docker.network.enable_ipv6: "true"
    ipam:
      driver: default
      config:
        - subnet: 172.16.238.0/24
          gateway: 172.16.238.1
        - subnet: 2001:1234:5678::/64
          gateway: 2001:1234:5678::1
```

## 7、配置新的 IPv6 NAT

``` shell
sudo ip6tables -t nat -A POSTROUTING -s 2001:1234:5678::/64 ! -o docker0 -j MASQUERADE
```

## 8、持久化 ip6tables

Debian 系可以使用 iptables-persistent 包来进行持久化，比如 Debian 11：

``` shell
sudo apt install iptables-persistent
```

并确保服务是开机自启动的：

``` shell
sudo systemctl is-enabled netfilter-persistent.service
```

如果不是，可以执行：

``` shell
sudo systemctl enable netfilter-persistent.service
```

查看对应服务状态：
``` shell
sudo systemctl status netfilter-persistent.service
```

为了保证重启后，还能自动加载当前的规则：

``` shell
sudo iptables-save > /etc/iptables/rules.v4
OR
sudo ip6tables-save > /etc/iptables/rules.v6
```

RHEL/CentOS 系可以使用 iptables-services，一般命令如下：

``` shell
# Disable firewalld if installed #
sudo systemctl stop firewalld.service
sudo systemctl disable firewalld.service
sudo systemctl mask firewalld.service
# install package on Linux to save iptables rules using the yum command/dnf command ##
sudo yum install iptables-services
sudo systemctl enable iptables
sudo systemctl enable ip6tables
sudo systemctl status iptables
```