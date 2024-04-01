---
layout: post
title: "2024 年的 vagrant 使用体验"
date: 2024-04-01 10:26:20 +0800
categories: ["2024"]
tags: [vagrant, thought]
---

「在我这是好的，不信你来看看」这是个程序员都明白的梗，背后反映出来的其实是软件开发环境与运行环境不一致的问题，基础设施、操作系统、版本、配置的差异都有可能导致异常情况的产生。避免各个环境的差异的关键点在于实现软件交付的可重复性，简单而言就是将每个步骤都像流水线一样记录下来形成脚本，这样无论脚本执行多少次，其结果都是可预测的。那么如何实现这种可重复性呢？

按照我的经验大概总结出以下：

- 避免人为因素干扰，禁止 SSH 连接服务器进行手工操作引入的差异
- 通过基础设施即代码（Infrastructure as Code）保证所有的软件部署环境一致
- 通过配置分离将软件在不同环境的运行差异剥离出来
- 通过 CI/CD 将构建交付流程自动化

而这些需求都催生出了 Docker 这样的解决方案，通过 Dockerfile 定义的镜像就保证了操作系统以及运行环境的一致性，通过环境变量的方式改变软件配置，精简的镜像有利于应用的快速交付。

时间回到十年前，我的运维同事是使用 Ansible 或者 Puppet 之类的工具进行自动化部署，这些工具需要知道服务器的密码，建立 SSH 连接执行部署脚本。也有通过制作虚拟机镜像的方式进行交付，一个镜像中包含了所有需要的软件，例如 LAMP（Linux，Apache，Mysql，Python）环境以及应用，与后来 Docker 镜像的思路如出一辙，只不过「one process per container」的思想决定了 Docker 镜像更加轻量。还有就是 Vagrant，通过编写 Vagrantfile（与 Dockerfile 的命名都如此一致）将安装步骤脚本化，在开发环境本地创建一台虚拟机，保证所有人的软件开发运行环境都是相同的，因为 Vagrant 的虚拟化层（Hypervisor）支持 VirtualBox，Hyper-V，因此在 Mac 或者 Windows 环境下都能得到一致的体验。

在 Docker 无处不在的 2024 年，为什么还依然选择使用 Vagrant 呢？

举个例子，这是我在 VictoriaMetrics 二次开发过程中，我想需要的开发环境：

- 可以有一个快速启动销毁的环境，里面包含了开发过程中所需要的依赖，例如需要部署好特定版本的 
VictoriaMetrics
- 可以通过 SSH 连接进入，执行操作命令，查看应用运行状态，例如执行 `vmctl` 命令
- 可以同步分发给开发成员，通过代码仓库管理 Vagrantfile，管理版本差异

一开始我也选择了官方提供的 docker-compose 部署方案，但是实际开发过程中发现体验实在是不如虚拟机直接。对比而言，我认为 Vagrant 通过虚拟机的方式对开发测试更加友好，而 Docker 则更适用于应用的交付。

最后，Vagrant 启动集群版本 VictoriaMetrics 的代码仓库：

<https://github.com/zhengtianbao/vagrant-victoriametrics.git>
