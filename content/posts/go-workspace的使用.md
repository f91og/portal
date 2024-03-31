+++
title = 'Go Workspace的使用'
date = 2024-02-28T14:58:32+09:00
draft = false
tags = ['Go', '依赖管理']
categories = ['Golang']
+++

最近项目中用到了go workspace，以前就知道这个，不过没怎么研究过，现在来记录下go workspace的使用。
<!--more-->

go workspace是go语言中管理多个项目的一种方式。go workspace包含一个或多个go module,每个module都有自己的代码和依赖。我这个项目中使用go workspace的原因是项目A用到了我们组自己的项目B，如果不用workspace则项目A的开发每次都需要项目B的正式发版才能用最新的项目B中的代码，但是用worksapce就可以将开发中的项目B的代码作为本地代码来让项目A用，非常方便（在go.mod中使用replace将远程的repo替换为本地repo的路径也可以，但是没有go workspace方便）。

**需要go版本1.18以上才能使用workspace**

## 基本的使用流程
**1. 初始化workspace**  
先建个单独的目录（目录名随意，没有固定要求），然后初始化这个目录为go workspace目录: 
```shell
mkdir workspace && cd workspace
go work init
```
会在workspace下生成一个`go.work`文件

**2. 在workspace下添加项目**  
在workspace目录下，用`go mod init`来新建一个项目（module）或者是clone一个项目然后将其加入到workspace里，我这里是clone一个项目
```shell
git clone repoB
go work use repoB
```

**3. 将项目A也clone到workspace下，并添加到workspace里**  
```shell
git clone repoA
go work use repoA
```
现在`go.work`文件里的内容是这样的:  
```go
go 1.20

use (
	./repoB
	./repoA
)
```
go.work 文件的语法和 go.mod 类似（go.work 优先级高于 go.mod），因此也支持 replace。

注意：实际项目中，多个模块之间可能还依赖其他模块，建议在 `go.work` 所在目录执行 `go work sync`

**5. 在项目A中编辑`go.mod`，直接像一般那样引用repoB即可**
```go
require (
	github.com/f91og/repoB v0.1.3
	github.com/bndr/gojenkins v1.1.0
    ......
)
```
go.work 不需要提交到 Git 中，因为它只是你本地开发使用的。当repoB开发完成，应该先提交其到远程仓库，然后在repoA中执行 `go get -u github.com/f91og/repoB`，然后禁用 workspace（export GOWORK=off 或者 GOWORK=off go run main.go ），再次运行 repoA 模块来最终测试更新后的远程仓库repoB中的代码是否正确。

## 在vscode中使用go workspace
最新的vscode的go插件已经支持了go workspace，点击在编辑go代码时状态栏出现的那个go版本就可以跳出 `open go.work` 的弹窗，然后可以直接编辑
![](/images/image.png)


虽然不清楚go workspace的底层是怎么实现的，但是估计是让go在寻找依赖时，优先去查找本地的吧。


本文参考了 https://go.dev/doc/tutorial/workspaces, https://polarisxu.studygolang.com/posts/go/workspace/


