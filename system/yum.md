# 配置 YUM 源

集群配置 YUM 源主要分为两种方式：使用镜像服务器和使用本地源。主要目的都是加快 `yum` 命令安装软件包的速度，甚至断网时也能够安装。

### 使用镜像服务器

该方式和配置个人计算机 YUM 镜像的步骤相同。主要是修改 `/etc/yum.repos.d/CentOS-Base.repo` 文件，在各个镜像服务器网站上已经有比较详细的说明。以[清华镜像](https://mirrors.tuna.tsinghua.edu.cn/help/centos/)为例，修改后该文件应该类似这样（如果你要用清华源的话可以直接复制）：

```text
[base]
name=CentOS-$releasever - Base
baseurl=https://mirrors.tuna.tsinghua.edu.cn/centos/$releasever/os/$basearch/
#mirrorlist=http://mirrorlist.centos.org/?release=$releasever&arch=$basearch&repo=os
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7

#released updates
[updates]
name=CentOS-$releasever - Updates
baseurl=https://mirrors.tuna.tsinghua.edu.cn/centos/$releasever/updates/$basearch/
#mirrorlist=http://mirrorlist.centos.org/?release=$releasever&arch=$basearch&repo=updates
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7

#additional packages that may be useful
[extras]
name=CentOS-$releasever - Extras
baseurl=https://mirrors.tuna.tsinghua.edu.cn/centos/$releasever/extras/$basearch/
#mirrorlist=http://mirrorlist.centos.org/?release=$releasever&arch=$basearch&repo=extras
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7

#additional packages that extend functionality of existing packages
[centosplus]
name=CentOS-$releasever - Plus
baseurl=https://mirrors.tuna.tsinghua.edu.cn/centos/$releasever/centosplus/$basearch/
#mirrorlist=http://mirrorlist.centos.org/?release=$releasever&arch=$basearch&repo=centosplus
gpgcheck=1
enabled=0
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7

```

注意 30 行的地方 `enabled=0` 表示不启用 `centosplus` 这一组库。经过这样设置，`base` `updates` `extras` 就会从清华源安装了。利用同样的方式还可以配置 `epel` 等其它源，此处不再赘述。

{% hint style="warning" %}
只有能访问到镜像服务器的节点才能使用这种方式。如果没有配置[计算节点连接外网](https://ryanbernx.gitbook.io/cluster-admin/system/network#ji-suan-jie-dian-lian-jie-wai-wang-nat)，那么计算节点将不能使用这种方式安装软件。
{% endhint %}

### 使用本地源

本地源就是先将 CentOS 软件安装包使用某种方式存放在集群的某个节点上，之后通过修改 `/etc/yum.repos.d` 中配置文件的方式让各个节点从本地获取安装包。由于安装文件在本地，因此计算节点在不联网的情况下也可以安装。常用的构建本地源的方式是复制已有镜像源或使用 ISO 镜像。

#### 复制已有镜像

使用以下脚本可以将清华源中 CentOS 7 相关包下载到本地：

{% code title="sync\_centos.sh" %}
```bash
#!/bin/bash
#
# from TUNA

RSYNC="rsync --delete -amP"
# 为了文明下载，可以限制自己下载速度为 2000 KBps
#RSYNC="rsync --delete -amP --bwlimit=2000"

# CentOS 7
$RSYNC \
    --include='7*' \
    --include='7*/**' \
    --exclude='*' \
    rsync://mirrors.tuna.tsinghua.edu.cn/centos/ /opt/centos

```
{% endcode %}

同步完毕后会在 `/opt/centos` 下看到类似如下结构：

```text
- 7（软连接，指向最新小版本对应的目录）
- 7.0.1406
- 7.1.1503
- ...
```

{% hint style="warning" %}
CentOS 系列每经过一个小版本更新，源中的安装包就会自动移动到最新版本的目录下。例如从 7.8 更新到 7.9，则同步完毕后 7.8 对应的目录为空。
{% endhint %}

{% hint style="danger" %}
以上脚本只针对设置了 rsync 协议的 CentOS 源有效（不一定选择清华的源，最好选择离自己比较近的服务器），详细信息请查看[官方文档](https://wiki.centos.org/HowTos/CreateLocalMirror)。
{% endhint %}

同理，epel 软件源也可以这样复制，详见[官方文档](https://fedoraproject.org/wiki/Infrastructure/Mirroring)。

由于镜像服务器经常会更新，复制的本地镜像最好也需要和上游同步。可以使用 crontab 来定时进行同步。输入命令 `crontab -e` 打开 crontab 的编辑界面，之后添加如下一行：

```text
10 4 * * * /root/bin/sync_centos.sh > /tmp/rsync-centos.log 2>&1
```

以上假定了同步的脚本放在了 `/root/bin` 中，且使用 root 用户操作。设置成功后，相应节点就会在每日 4:10 左右与上游镜像服务器同步一次。

#### 使用 ISO 镜像

如果已经有 CentOS 的 ISO 文件，也可以不同步镜像服务器，直接将 ISO 文件挂载到本地。可以获取 ISO 文件的镜像列表参考[这里](http://isoredirect.centos.org/centos/7/isos/x86_64/)。需要注意，只有完整版 ISO（后缀形如 `Everything-XXXX.iso`）才包含全部的安装包，大小约为 10 GB。

下载完毕后，挂载 ISO 文件：

```text
mkdir -p /opt/centos/7
# 以下文件名替换成实际的值
mount -o loop CentOS-7-x86_64-Everything-XXXX.iso /opt/centos/7
```

之后在 `/opt/centos/7` 目录下会看到和镜像中类似的结构。

{% hint style="danger" %}
epel 仓库没有提供 ISO，因此不能使用这种方式创建本地 epel 源，只能通过复制镜像服务器的方式。
{% endhint %}

#### 将本地源共享给其它节点

无论是使用镜像服务器还是 ISO，下载的文件只在一个节点上存在（登录节点 or 存储节点），想要使得计算节点可以访问这些目录必须配置网络文件系统（NFS），我们在[配置 NFS](https://ryanbernx.gitbook.io/cluster-admin/system/nfs) 部分介绍全部过程。

#### 修改 YUM 配置文件

本地存在 CentOS 源后，将 YUM 的配置指向本地目录，方式还是修改 `CentOS-Base.repo` 文件。

```text
[base]
name=CentOS-$releasever - Base
baseurl=file:///opt/centos/$releasever/os/$basearch/
#mirrorlist=http://mirrorlist.centos.org/?release=$releasever&arch=$basearch&repo=os
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7

#released updates
[updates]
name=CentOS-$releasever - Updates
baseurl=file:///opt/centos/$releasever/updates/$basearch/
#mirrorlist=http://mirrorlist.centos.org/?release=$releasever&arch=$basearch&repo=updates
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7

#additional packages that may be useful
[extras]
name=CentOS-$releasever - Extras
baseurl=file:///opt/centos/$releasever/extras/$basearch/
#mirrorlist=http://mirrorlist.centos.org/?release=$releasever&arch=$basearch&repo=extras
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7

#additional packages that extend functionality of existing packages
[centosplus]
name=CentOS-$releasever - Plus
baseurl=file:///opt/centos/$releasever/centosplus/$basearch/
#mirrorlist=http://mirrorlist.centos.org/?release=$releasever&arch=$basearch&repo=centosplus
gpgcheck=1
enabled=0
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7

```

其实就是把 `baseurl=` 的区域换成了本地目录，之后 YUM 就会到本地源中下载了。

### 总结

这里介绍了两种配置 YUM 源的方法，方便集群快速安装软件。

* 使用镜像服务器：
  * 修改 `CentOS-Base.repo` 和 `epel.repo` 等文件。
  * 需要所有节点配置访问外网。
* 使用本地源：
  * 复制已有镜像或使用 ISO 文件。
  * （可选）使用 crontab 配置每日的同步。
  * 将本地源以网络文件系统的方式共享给集群（见[配置 NFS](https://ryanbernx.gitbook.io/cluster-admin/system/nfs)）。
  * 修改 `CentOS-Base.repo` 和 `epel.repo` 等文件。

