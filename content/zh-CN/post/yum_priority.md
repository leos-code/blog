
+++
title = "安装librdkafka-devel 遇到的yum源优先级问题"
date = "2022-07-19"
description = "安装librdkafka-devel 遇到的yum源优先级问题"
tags = [
    "yum", "confluent-kafka-go"
]
+++


# 安装librdkafka-devel 遇到的yum源优先级问题

## 事情起因
公司有个golang项目依赖`confluent-kafka-go`。

这个包是 `librdkafka`(c++) 项目的golang绑定，所以需要在机器上安装正确的librdkafka包项目才能编译成功

我们服务是aws linux 2的机器，相当于centos7。
按照文档说明配置yum 源：
```
sudo rpm --import https://packages.confluent.io/rpm/7.2/archive.key

#创建/etc/yum.repo.d/confluent.repo，内容如下:
[Confluent]
name=Confluent repository
baseurl=https://packages.confluent.io/rpm/7.2
gpgcheck=1
gpgkey=https://packages.confluent.io/rpm/7.2/archive.key
enabled=1

[Confluent-Clients]
name=Confluent Clients repository
baseurl=https://packages.confluent.io/clients/rpm/centos/$releasever/$basearch
gpgcheck=1
gpgkey=https://packages.confluent.io/clients/rpm/archive.key
enabled=1
```
> aws linux 2 的 $releasever 是2，需要手动改为7(centos7)

配置好后，执行`yum list librdkafka-devel` 没有发现最新的1.9.0的包，很疑惑。
> confluent-kafka-go的依赖是1.9.0，必须安装librdkafka-devel 1.9.0才能编译成功。
> aws 自带的librdkafka-devel 版本太低

## 排查过程
* 打开
(https://packages.confluent.io/clients/rpm/centos/7/x86_64/)[] 是可以找到`librdkafka-devel-1.9.0-1.cflt.el7.x86_64.rpm`rpm包，说明yum repo的地址没问题。

* 下载`librdkafka-devel-1.9.0-1.cflt.el7.x86_64.rpm` rpm文件，手动 安装`rpm -i librdkafka-devel-1.9.0-1.cflt.el7.x86_64.rpm` 提示缺少另外的依赖rpm。rpm 安装需要自己维护依赖，放弃rpm 安装。

* 在google搜这个问题关键词，找的内容都不匹配，随后放弃直接搜索关键词，准备全面学一下yum的基础知识

* 在redhat官网文档中找到了一个全面的yum命令文档，每个命令都有简单的介绍：[yum 命令备忘录](https://access.redhat.com/sites/default/files/images/yumcheat_01_0.png)。 

* 看文档后尝试并发现`yum localinstall librdkafka-devel-1.9.0-1.cflt.el7.x86_64.rpm`  可以安装本地rpm并自动下载依赖。(localinstall 后面可以指定url 安装）

成功，悲喜交加，搞了2个小时解决了。

但yum list 或 yum search 为什么没找到 librdkafka-devel 包的问题还是找到

## 追查问题并找到原因

看着yum备忘录上的命令，尝试执行`yum repoinfo Confluent-Clients/x86_64`命令查看repo的信息。

突然注意到了提示信息：`yum 68 packages excluded due to repository priority protections`

在google搜了这个报错，找到了问题的原因：
**yum源的优先级** —— 如果yum中多个源都包含同一个包时，它只会取最高优先级的源的包

给`/etc/yum.repo.d/confluent.repo`添加 `priority=1`，设置第一优先级(priority N越大优先级越小，取值1到100)

执行 `yum list librdkafka-devel-1.9.0-1.cflt.el7.x86_64.rpm`，`1.9.0-1` 版本可以找到了
> .修改priorities的配置文件/etc/yum/pluginconf.d/priorities.conf，enabled=0 可以禁用priorities插件。也可以解决优先级问题 (不过我没试过)


