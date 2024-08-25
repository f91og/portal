+++
title = 'go get 私有仓库'
date = 2024-05-21
draft = false
toc = false
tags = ['go']
categories = ['Golang']
+++

最近开发中需要使用公司自己的bitbucket仓库，直接 go get 报错，还是要设置一些东西。

<!--more-->

首先公司的代码仓库是托管在自己的 bitbucket 上的，这点就比较麻烦。直接尝试 `go get git.my-company.com/repoA` , 但是不行，需要配置一些东西

首先是确认可以连接到私有的bitbucket
编辑 `` `~/.ssh/config` ``， 配置自己公司的 bitbucket host 和端口
```
Host git.my-company.com

Port 7999
```

然后添加 key
[Creating SSH keys | Bitbucket Data Center 7.21 | Atlassian Documentation](https://confluence.atlassian.com/bitbucketserver0721/creating-ssh-keys-1115665672.html?utm_campaign=in-app-help&utm_medium=in-app-help&utm_source=stash)

bitbucket连接没问题后，可以配置 `gitconfig` ，自动将https的clone转为ssh的
```
[url "git@git.my-company.com:"]
	insteadOf = https://git.my-company.com/
```

最后就是要配置 `GOPRIVATE` 这个环境变量：
```shell
export GOPRIVATE="git.my-company.com"
```

最后在需要使用私有repo的 module 中执行 `go get git.my-company.com/repoA` ` 就可以了。

当然使用 go workspace 也可以实现在自己的项目中使用私有仓库，而且更加简单方便。