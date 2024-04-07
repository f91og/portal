+++
title = '使用hugo和github pages来搭建自己的博客'
date = 2024-02-22T00:22:52+08:00
draft = false
toc = true
tags = ['Blog', 'Content']
categories = ['Golang']
+++

最近还是想把笔记上的东西弄到博客，一是在网上分享有成就感，二是也方便自己参考，记录下博客的搭建过程，基本上照着hugo的官方文档来就可以，不需要看些参差不齐的博客浪费时间。 
<!--more-->

## 1. hugo的安装，基本使用，命令
看这里：https://gohugo.io/getting-started/quick-start/
```shell
# create a new pst
hugo new content posts/new-blog.md 

# create a new blog directory, no need to create dir first, hugo will automatically create
hugo new content posts/new-blog/index.md 
```

## 2. 部署到github page并用github action来自动部署
看这里：https://gohugo.io/hosting-and-deployment/hosting-on-github/

## 3. 处理博文中有本地图片的情况
把图片放到static文件夹下，然后在博文中引用static目录下的图片文件的路径, eg: `![](/images/example.png)`
![](/images/example.png)

第二种方式是在`content/posts`目录下创建一个文件夹，在里面创建`index.md`文件，把博客内容放到里面，然后把图片文件也放到这个文件夹下，在博客里直接相对路径引用这个图片就可以了。
https://stackoverflow.com/questions/71501256/how-to-insert-an-image-in-my-post-on-hugo

## 4. 解决目录多语言不匹配和文本不完全显示
在使用这个主题 `maupassant` 这个主题时出现了2个问题，一个是目录的标题显示成了英文 `TABLE OF CONTENT` ，另一个是标题长度超过目录栏目宽的时候没有不能完全显示。

**目录标题的中英文问题：**  
在项目根目录的 `toml` 配置文加里，设置如下：
```toml
defaultContentLanguage = "zh-hans"
```
但是这样目录就直接没标题了，反而更好。

**目录文本长度过长不能完全显示：**  
这个就有点麻烦了，貌似需要改主题的 `css`
找到主题目录下的 `static/layouts/partials/toc.html` 文件，修改其中的 `.post-toc-content` 样式，修改完了 `hugo server -D` 会自动sync，不需要任何重启或者刷新浏览器。
```css
...
    .post-toc .post-toc-content {
        font-size: 15px;
        white-space: normal;
        overflow-wrap: break-word; /* 或者使用 word-wrap: break-word; */
    }
...
```

备注：  
浏览器调试里那些被划横线的样式说明被其他样式覆盖了，[使用chrome调试工具解决问题（一） - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/624465440)

调整完成后又有新的问题，这个主题的默认样式左侧栏空间太小了，最好是把目录给移到右下方。

然后需要用本地 `theme` 文件夹来替换 `git submodule`，要不然还得fork原主题提交修改才行，很麻烦。
1. 在博客根目录里的 `.git/config` 里删除 `submodule` 的相关配置
2. 删除 `.gitmodules` 文件
3. `git add themes` 将主题文件夹添加到本地仓库

## 5. 让首页显示简短摘要
使用 `maupassant` 主题时，在要作为摘要的内容的后面加上这个 `<!--more-->`

## 6. 自定义域名
最后买个域名替换下github page的域名就可以了，.com的域名一个也不是很贵