---
title: "golang交叉编译"
date: 2020-05-24
draft: false
tags: ["golang"]
categories: ["技术"]
keywords: ["golang", "交叉编译", "golang交叉编译", "Go语言跨平台编译", "多平台构建", "go build", "目标平台", "跨平台开发"]
description: "golang在不同平台编译其他架构和系统的方法"
---

# golang交叉编译

## 关于在linux和mac下交叉编译其他平台
```
CGO_ENABLE=0 GOOS=linux GOARCH=amd64 go build main.go
```
```
CGO_ENABLE=0 GOOS=darwin GOARCH=amd64 go build main.go
```
```
CGO_ENABLE=0 GOOS=windows GOARCH=amd64 go build main.go
```

## 关于windows上交叉编译其他平台
cmd下
```cmd
SET CGO_ENABLE=0 
SET GOOS=linux 
SET GOARCH=amd64 
go build main.go
```

## 全部
可以go1.13后可以使用工具链
```
go env -w GOOS=linux GOARCH=amd64 CGO_ENABLED=0
```

## 总结
GOOS: darwin freebsd linux windows
GOARCH: 386 amd64 arm
交叉编译不支持CGO(windows)
其实就先设置临时环境变量在编译
