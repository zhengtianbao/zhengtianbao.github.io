---
layout: post
title: "kubernetes cookbook 之项目结构篇"
date:  2021-10-13 14:22:29
categories: Kubernetes
---

![go](/images/go.jpeg)

写在前面：本系列是在阅读 kubernetes 源码过程中逐渐整理的一些值得借鉴的编程范式（recipe），汇集起来也就有了这个 **cookbook**，所以内容展示的更多的可能是 code snippet，而不是 How to use Kubernetes。

## Kubernetes Project Layout

Go 语言社区提供了一套标准的工程目录结构方案，详细参考：<https://github.com/golang-standards/project-layout>

Kubernetes 项目结构也是逐步迁移符合至社区标准，目录结构说明如下所示：

|目录|说明|
|:---- | :---- |
|api/|存放OpenAPI/Swagger的spec文件，包括Json，Protocol的定义|
|build/|构建相关脚本|
|CHANGELOG/|版本变动文件|
|cmd/|可执行文件入口，例如kubelet，kube-proxy|
|docs/|文档|
|hack/|构建，测试相关的脚本|
|logo/|各种格式尺寸的logo图片|
|pkg/|内部包|
|plugin/|插件代码目录，例如认证，授权|
|staging/|暂存目录，大多软链接到vendor/k8s.io目录|
|test/|测试相关|
|third_party/|第三方工具，代码|
|vendor/|依赖的代码库|

假设我想新建一个 kubernetes 风格的项目暂时命名为 canoe 吧，它是一个简单的 http server，启动的时候可以接收参数指定绑定的端口号，那么就先生成以下目录文件：

```
➜  canoe tree
.
├── build
│   └── Makefile
├── cmd
│   └── server
│       ├── options
│       │   ├── options.go # 命令行参数
│       │   └── validation.go # 验证命令行参数
│       └── server.go # 命令入口
├── Makefile -> build/Makefile
├── pkg
│   └── server
│       └── server.go # 业务逻辑
└── README.md

6 directories, 7 files

```

这样就初始化了一个只有骨架（skeleton）的工程，后续将逐步填充。
