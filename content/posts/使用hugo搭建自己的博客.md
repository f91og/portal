+++
title = '使用hugo和github pages来搭建自己的博客'
date = 2024-02-22T00:22:52+08:00
draft = false
toc = true
tags = ['go', 'how-to']
categories = ['Golang']
+++

最近还是想把笔记上的东西弄到博客，一是在网上分享有成就感，二是也方便自己参考，记录下博客的搭建过程，基本上照着hugo的官方文档来就可以，不需要看些参差不齐的博客浪费时间。 
<!--more-->

## 1. hugo的安装和基本使用
看这里：https://gohugo.io/getting-started/quick-start/
```shell
# mac上安装hugo
brew install hugo

# 创建自己的博客项目目录，确保之前本地已经安装好了Git
hugo new site myblogs
cd myblogs
git init

# 使用hugo命令来新建一篇博文，自己手动在posts目录下新建一个markdown文件也可以
hugo new content posts/new-blog.md 

# create a new blog directory, no need to create dir first, hugo will automatically create
hugo new content posts/new-blog/index.md 

# 在本地实时预览博客效果，会自动sync修改
hugo server -D
```
## 2. 部署到github page并用github action来自动部署
看这里：https://gohugo.io/hosting-and-deployment/hosting-on-github/

## 3. 处理博文中有本地图片的情况
把图片放到博客目录中的`static`目录下，然后在博文中引用`static`目录下图片文件的路径, eg: `![](/images/example.png)`
![](/images/example.png)

第二种方式是在`content/posts`目录下创建一个目录单独用于一篇博文，在里面创建`index.md`文件作作为博文内容，把图片文件也放到这个目录下，在`index.md`里直接相对路径引用这个图片就可以了。
https://stackoverflow.com/questions/71501256/how-to-insert-an-image-in-my-post-on-hugo

## 4. 解决目录多语言不匹配和文本不完全显示
在使用这个主题 `maupassant` 这个主题时出现了2个问题，一个是目录的标题显示成了英文 `TABLE OF CONTENT` ，另一个是标题长度超过目录栏目宽的时候没有不能完全显示。

**目录标题的中英文问题：**  
在项目根目录的 `toml` 配置文加里，设置如下：
```toml
defaultContentLanguage = "zh-hans"
```

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

最后就是将本地的 `theme` 目录来替换 `git submodule`，否则需要fork原主题git仓库提交修改后才能让改动的样式生效，很麻烦。
1. 在博客根目录里的 `.git/config` 里删除 `submodule` 的相关配置
2. 删除 `.gitmodules` 文件
3. `git add themes` 将本地的 `theme` 目录托管到 git

## 5. 让首页显示简短摘要
使用 `maupassant` 主题时，在要作为摘要的内容的后面加上这个 `<!--more-->`

## 6. 换行问题
markdown 换行有点麻烦，普通段落想要换行的话直接敲回车是没有用的，连续输入2个空格可以换行，但是这个以代码段结尾没有用，可以输入**一个**反斜杠符号 \\ 来解决这个问题。

## 7. 自定义域名
这步折腾了一会儿，搞不定时不要瞎尝试，要仔细思考各种可能行并且阅读官方文档，从弄懂基本的原理开始。

**购买域名并配置**  
在nameslio上买了个域名，按照 github pages 的官方文档的说明，在nameslio上配置dns，添加一个CNAME记录，将自己的域名和 user.github.io 对应起来，**并且A记录也要按照文档上说的来添加**，否则custom domain检查的时候会报 `Domain does not resolve to the GitHub Pages server` 错误，被这个坑惨了。

**github pages的配置**  
在自己的静态网站的 github repo 页面的 Setting 中的 Pages 里配置 Custom domain（注：需要先在 Build and deployment 里选择 Github Actions 然后才会出现Custom domain选项，将带www子域名的完整域名填入custom domain里）
[管理 GitHub Pages 站点的自定义域 - GitHub 文档](https://docs.github.com/zh/pages/configuring-a-custom-domain-for-your-github-pages-site/managing-a-custom-domain-for-your-github-pages-site#about-custom-domain-configuration)

**Domain does not resolve to the GitHub Pages server**  
配置完成后总是有这个问题，捣鼓了半天解决不了，原来是A record和CNAME record都要配置才可以，我真的是裂开。
[dns - How to fix: Domain does not resolve to the GitHub Pages server. Error in Github Pages for custom domain setup with Enforce HTTPS Enabled? - Stack Overflow](https://stackoverflow.com/questions/54059217/how-to-fix-domain-does-not-resolve-to-the-github-pages-server-error-in-github)

整个流程走下来还是花了很多时间，遇到问题不知道怎么办怎么搞定的时候第一步肯定是认真读完整资料，注意是读完整，别读了一点就瞎捣鼓，然后重要的是在读完资料后仔细思考可能的原因，什么都不去想找到一点是一点的瞎弄最浪费时间和精力。