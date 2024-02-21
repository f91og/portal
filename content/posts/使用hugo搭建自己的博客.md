+++
title = '使用hugo和github pages来搭建自己的博客'
date = 2024-02-22T00:22:52+08:00
draft = true
toc = true
+++

最近还是想把笔记上的东西弄到博客，一是在网上分享有成就感，二是也方便自己参考，记录下博客的搭建过程，基本上照着hugo的官方文档来就可以，不需要看些参差不齐的博客浪费时间。

# 1. hugo的安装，基本使用，命令
看这里：https://gohugo.io/getting-started/quick-start/

# 2. 部署到github page并用github action来自动部署
看这里：https://gohugo.io/hosting-and-deployment/hosting-on-github/

# 3. 如何处理博文中有本地图片的情况
把图片放到static文件夹下，然后在博文中引用static目录下的图片文件的路径, eg: `![](/images/example.png)`
![](/images/example.png)
好像还有别的方式，改天研究下

最后买个域名替换下github page的域名就可以了，.com的域名一个也不是很贵