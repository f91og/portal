+++
title = ' 关于 DNS 的 6 个基础问题 '
date = 2024-05-01
draft = false
toc = false
tags = ['network', 'dns']
categories = ['Infrastructure']
+++

在配置博客的自定义域名时遇到了一堆 DNS 配置的问题，其实不是什么高级的 DNS 相关的问题，只是一直以来没有好好整理，现在记录下这些问题。
<!--more-->

## 1. DNS 配置的基本作用
将域名映射为一个 ip 地址，或将域名映射为另一域名等等，让浏览器访问域名时能够正确的访问到服务器的 ip 地址，当然还有别的一些用法。
基本工作原理就是当本地找不到域名的 ip 映射时，去请求域名解析服务器（DNS 服务器），拿到域名的 ip。

## 2. Linux 上 DNS 如何配置
注意这里的配置是一般机器上的配置，不是 DNS 服务器的配置。
有 2 个配置，一个配置 DNS 服务器的 ip，另一个是本地 DNS 解析记录的配置。

域名解析服务器的 ip 配置在 `/etc/resolv.conf` 里
```sh
# vi /etc/resolv.conf

nameserver 198.19.248.200

# 启用 EDNS 版本 0，它允许在 DNS 解析过程中传输更大的数据包，以支持更大的有效负载和更高的性能
options edns0 

# 指定在进行主机名解析时的搜索路径，ping example 时不会给example后加后缀
# 反之如果是 search com 的话，会自动加上com后缀，变成 example.com
search .
```

本地 DNS 解析记录的配置在 `/etc/hosts` 里，当请求的域名的 ip 不在这里时就会向 DNS 服务器请求域名的 ip
```sh
# vi /etc/hosts

127.0.1.1       ubuntu01
127.0.0.1       localhost
::1             localhost ip6-localhost ip6-loopback
ff02::1         ip6-allnodes
ff02::2         ip6-allrouters
```

## 3. CNAME，A，TXT 记录是什么
首先明确一点这些都是在**域名解析服务器**（DNS 服务器）上配置的，而不是在一般主机上的 `/etc/hosts` 文件中配置的。

CNAME（Canonical Name Record），真实名称记录，**是域名系统（DNS）的一种记录**。 CNAME 记录用于将一个域名（同名）映射到另一个域名（真实名称），域名解析服务器遇到 CNAME 记录会以映射到的目标重新开始查询。 这对于需要在同一个 IP 地址上运行多个服务的情况来说非常方便。

A record，即 address record，一条记录对应一个域名和 ip 地址。

DNS“文本”(TXT) 记录允许域管理员将文本输入到域名系统 (DNS) 中。TXT 记录最初的目的是用作存放人类可读笔记的地方。但是，现在也可以将一些机器可读的数据放入 TXT 记录中。一个域可以有许多 TXT 记录

DNS 服务器上这些记录配置的基本格式是这样
```shell
example.com. 　　IN 　　CNAME 　　websrv.heiye.com.
example.com. 　　IN 　　A 　　1.1.1.1
example.com.    IN    TXT   "v=spf1 mx -all"
```

## 4. 什么是 apex 域
An apex domain is a custom domain that does not contain a subdomain, such as example.com
简单来说就是 root domain，看域名要从右往左看，最右边最 root 的域名，其次是下一级。比如我买了 `f91og.com` 这个域名，那如果前面有 xxx.f91og.com，这个 xxx 就是子域名。
注：`www.f91og.com` 这里的 `www` 也被视为一个子域名。

## 5. DNSSEC 是什么
域名系统安全扩展 (DNSSEC) 是**域名系统 (DNS) 的一项功能，用于验证对域名查找的响应**。 它不会为这些查找提供隐私保护，但会阻止攻击者操控对 DNS 请求的响应或在该响应中投毒，也是在 DNS 服务器上配置的。

## 6. 如果在 DNS 记录上有个不存在的域名会发生什么
前公司在使用 CoreDNS 时总是会出现某个域名已经失效，但是业务团队的 app 中没有更新，一直向 CoreDNS 查询这条记录从而导致延迟。域名的查询是个递归查询，就是一直会找上面的 dns 服务器，直到找到后并缓存，或者上游找不到返回 NXDOMAIN，CoreDNS 对 `NXDOMAIN` 响应也会缓存一段时间（由 TTL 决定），但缓存失效后再次查询同一个不存在的域名仍会触发新的递归查询，增加不必要的网络请求负载。

