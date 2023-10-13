---
layout: post
title: golang环境配置
description: >
  VS code下使用golang生成exe和进行调试
slug: golang
sitemap: false
hide_last_modified: false
---

windows系统，使用vs code作为IDE，配置golang环境

## 下载

[Download and install - The Go Programming Language](https://go.dev/doc/install)

go1.21.3.windows-amd64

```
C:\Users\admin>go version
go version go1.21.3 windows/amd64
```

## 安装go插件

VS code搜索go，第一个

## 设置代理

cmd输入以下命令

```
go env -w GO111MODULE=on
go env -w GOPROXY=https://goproxy.cn,direct
```

go modules功能的开关是GO111MODULE

https://goproxy.cn是国内代理

在使用go modules时，GOPATH是无意义的

> The "gopls" command is not available. Run "go install -v golang.org/x/tools/gopls@latest" to install.

## 下载依赖

VS中ctrl shift p搜索go install/update tools

> ctrl K T 选择VSC主题

下载所有依赖

![image-20231011203513207](https://raw.githubusercontent.com/lanyily/Cplusplus-pic/main/2023/202310112035245.png)

## go modules依赖管理

```
go mod tidy
```

将当前源码文件所依赖的包安装，多的删掉，少了补上

```
go mod init 应用名称(项目名称)  
```

执行该命令，会在应用过的根目录下(家目录)，生成go.mod文件。

我们在进行go语言项目开发的时候，会依赖3种类型的库包：  

1. 内置的标准库包，在goroot/src目录下，也就是我们安装目录的src目录下(类似于python的bin目录)  
2. 第三方库包（git上开源的）  
3. 项目中的库包，也就是项目中的其他目录。自己调用自己写的函数方法

## 调试

文件目录

![image-20231011211014017](https://raw.githubusercontent.com/lanyily/Cplusplus-pic/main/2023/202310112110050.png)

打断点

![image-20231011210923999](https://raw.githubusercontent.com/lanyily/Cplusplus-pic/main/2023/202310112109056.png)
