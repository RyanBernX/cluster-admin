# 概述

这一部分主要介绍系统基础配置，有了这些才能在集群上搭建各种应用。需要注意的是，根据服务器制造商的情况，可能部分基础配置他们已经帮你完成，无需重复进行配置。但有时制造商什么也不管，这个时候就要真正动手从零开始组建集群。

为了指南的完整性，在这里会介绍所有的配置步骤。但并非每一个步骤都是必需的，需要看具体购买机器的场合。

### 购买独立服务器/工作站并直接使用

一般只有一台单独的机器，基本上运来插上电和网线就能用，制造商几乎什么都不用管（如果有机柜可能会帮你上架）。但是机器里面带的系统要么很老旧，要么直接是个 windows，还是要做一些操作。

* 安装操作系统/硬件驱动
* 配置硬盘分区
* 配置 SSH
* 系统安全策略

### 购买独立服务器并加入到已知集群

新机器还是买来就能用，但因为要加入到已知集群，必须要保持环境的一致性。此时需要：

* 安装操作系统/硬件驱动
* 配置硬盘分区
* 配置网络/hosts
* 配置 SSH
* 配置 NIS（客户端）
* 配置 NFS（客户端）
* 配置 NTP（客户端）
* 配置调度器（客户端）
* 系统安全策略

### 购买成套的集群

一般是较大规模的采购才会购入成套设备，买来需要经过工程师部署。此时一般不太需要操作系统、网络方面的部署。现在许多制造商都会推广他们的高性能计算平台，这种平台基本就是开源调度器外面套一层壳，可以直接使用。总的来说，需要：

* 了解网络/hosts
* 配置 NIS（可选）
* 配置 NFS（可选）
* 配置 NTP（可选）
* 配置调度器（可选）
* 系统安全策略

{% hint style="info" %}
我个人完全不建议选用制造商提供的傻瓜式平台，在能力允许范围内还是自己搭建比较好。一个原因是这些平台都需要额外收取费用，如果自己有技术完全可以省掉这笔钱。如果一定要安装此类平台，一些系统层面的设置可能已经含在平台中，建议向制造商索要详细的技术手册。
{% endhint %}



