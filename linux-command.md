---
layout: default
title: Linux软件安装命令详解
date: 2021-09-17 22:06
tags:
  - Linux
categories: [ tech, tutorial ]
---

# Linux软件安装命令详解

作为后端开发，平时在看博客以及学习时，经常遇到需要在Linux系统上进行部署、软件安装等操作，也遇到过很多安装命令，比如`yum`、
`apt-get`、`dpkg`等等，这些命令有什么区别？又有什么联系？独立开发时，如何确定使用哪一个？下面来详细说一说。下述内容出自《Linux命令行大全》
*William E.Shotts,jr*著。

## 软件包系统

不同的Linux发行版使用的时不同的软件包系统，并且原则上适用于一种的发行版的软件包和其他版本是不兼容的。大多数的Linux系统不外乎两种软件包：

* Debian
    * `.deb`技术
* Red Hat
    * `.rpm`技术

> 此处忽略Gentoo、Slackware和Foresight等

下面是两个软件包系统和其对应的软件包系统：

| 软件包系统   | 发行版本（部分）                                                           |
|---------|--------------------------------------------------------------------|
| Debian类 | Debian、Ubuntu、Xandros、Linspire                                     |
| Red Hat | Fedora、CentOS、Red Hat Enterprise Linux、openSUSE、Mandriva、PCLinuxOS |

可见，我们常用的CentOS、Ubuntu正好属于两个不同的软件包系统，正因为如此，后面我们会看到一些很眼熟的软件安装命令。

## 软件

首先是软件包，包文件是组成软件包系统的基本软件单元，它是由组成软件包的文件压缩成的文件集。这里面可能包含大量的程序以及相关的数据，或者安装脚本等等。包文件一般由开发者维护。成千上万的软件包集中起来就形成了
**库**。最后是最重要的**依赖关系**
，几乎没有哪个程序是独立的，程序之间相互依赖彼此，以完成既定工作。比如输入输出的工作就是由多个程序共享的例程执行。这些例程存储在共享库里面，共享库里面的文件为多个程序提供必须的服务。如果一个软件包需要共享库之类的共享资源，说明对其有
**依赖性**。显然共享库的存在就是为了解决多个程序都需要使用某个例程，如果安装时每个都带着这个必须的例程安装，那一定会出现安装包过大的问题。因此解决依赖性是一个很重要的能力，而现代软件包管理系统都提供依赖性解决策略。

## 高级和低级软件包工具

软件包管理系统通常包含两类工具：

1. 执行安装、删除软件包文件等任务的低级工具；
2. 进行元数据搜索及依赖性解决的高级工具。

下面是不同发行版本使用的高级、低级软件包系统工具

| 发行版本                                   | 低级工具 | 高级工具             |
|----------------------------------------|------|------------------|
| Debian 类（Ubuntu等）                      | dpkg | apt-get、aptitude |
| Fedora、CentOS、Red Hat Enterprise Linux | rpm  | yum              |

## 常见软件包管理任务

下面介绍一些常见的软件包管理操作，另外低级工具也支持软件包的创建（但我不会（是指我，不是指作者））。下述内容中`package_name`
指软件包的名称，`package_file`则是指包含在该软件包中的文件名。

### 在库里查找软件包

使用高级工具来搜索库元数据时，可以根据包文件名或其描述来查找该包：

| 系统类型       | 命令行                                         |
|------------|---------------------------------------------|
| Debian 系统  | apt-get<br />apt-cache search search_string |
| Red Hat 系统 | yum search search_string                    |

举例来说，在Red Hat系统的yum 库中搜索**emac**文本编辑器的代码如下：

`yum search emacs`

查找展示：

```shell
[asuka@localhost ~]$ yum search emacs
CentOS-8 - AppStream                                                  3.4 MB/s | 8.9 MB     00:02    
CentOS-8 - Base                                                       1.6 MB/s | 7.4 MB     00:04    
CentOS-8 - Extras                                                     2.9 kB/s |  10 kB     00:03    
=================================== Name & Summary Matched: emacs ====================================
emacs.x86_64 : GNU Emacs text editor
emacs-common.x86_64 : Emacs common files
emacs-filesystem.noarch : Emacs filesystem layout
emacs-filesystem.noarch : Emacs filesystem layout
emacs-nox.x86_64 : GNU Emacs text editor without X support
pinentry-emacs.x86_64 : Passphrase/PIN entry dialog based on emacs
emacs-terminal.noarch : A desktop menu item for GNU Emacs terminal.
emacs-lucid.x86_64 : GNU Emacs text editor with LUCID toolkit X support

```

### 安装库中的软件包

高级工具允许从库中下载、安装软件包，**同时安装所有的依赖包**

| 系统类型       | 命令行                                              |
|------------|--------------------------------------------------|
| Debian 系统  | apt-get update<br />apt-get install package_name |
| Red Hat 系统 | yum install package_name                         |

例如，在Debian系统下安装命令为：

* 先更新：`apt-get update`

* 再安装：`apt-get install emacs`

```shell
jungle@EVANGELION-01:~$ sudo apt-get install emacs
Reading package lists... Done
Building dependency tree       
Reading state information... Done
The following package was automatically installed and is no longer required:
  libllvm11
Use 'sudo apt autoremove' to remove it.
The following additional packages will be installed:
  emacs-bin-common emacs-common emacs-el emacs-gtk gsfonts imagemagick-6-common libfftw3-double3
  liblqr-1-0 libmagickcore-6.q16-6 libmagickwand-6.q16-6
Suggested packages:
  mailutils emacs-common-non-dfsg libfftw3-bin libfftw3-dev libmagickcore-6.q16-6-extra
The following NEW packages will be installed:
  emacs emacs-bin-common emacs-common emacs-el emacs-gtk gsfonts imagemagick-6-common
  libfftw3-double3 liblqr-1-0 libmagickcore-6.q16-6 libmagickwand-6.q16-6
0 upgraded, 11 newly installed, 0 to remove and 16 not upgraded.
Need to get 38.8 MB of archives.
After this operation, 144 MB of additional disk space will be used.
Do you want to continue? [Y/n] 
```

可见这里有安装依赖的说明。为后续讲解删除软件包，此处就先安装上。

### 安装软件包文件中的软件包

如果我们仅有一个软件包文件，那么可以使用低级工具进行安装，这样的安装显然并不安装依赖关系。

| 系统类型       | 命令行                         |
|------------|-----------------------------|
| Debian 系统  | dpkg --install package_file |
| Red Hat 系统 | rpm -i package_file         |

由于我手头并没有现成的软件包文件，下述例子出自书中：

`rpm -i emacs-22.1-7.fc7-i386.rpm`

显然Red Hat 体系下的软件包后缀为`.rpm`。

> 注意，由于是采用低级工具安装，所以并不会解决依赖问题，一旦安装过程缺少依赖包，rpm就会抛出错误后退出。

### 删除软件包

卸载软件既可以使用高级工具也可以使用低级工具，高级工具的命令如下

| 系统类型       | 命令行                        |
|------------|----------------------------|
| Debian 系统  | apt-get rmove package_name |
| Red Hat 系统 | rpm erase package_name     |

下面卸载上文安装的emacs：

`apt-get remove emacs`

```shell
jungle@EVANGELION-01:~$ sudo apt-get remove emacs
...
dpkg: warning: files list file for package 'libgeocode-glib0:amd64' missing; assuming package has no files currently installed
dpkg: warning: files list file for package 'laptop-detect' missing; assuming package has no files currently installed
...
(Reading database ... 1402 files and directories currently installed.)
Removing emacs (1:26.3+1-1ubuntu2) ...
```

### 更新库中的软件包

| 系统类型       | 命令行                              |
|------------|----------------------------------|
| Debian 系统  | apt-get update ; apt-get upgrade |
| Red Hat 系统 | rpm update                       |

过于直白，此处不再演示。

### 更新软件包文件中的软件包

与上文安装相对应：

| 系统类型       | 命令行                         |
|------------|-----------------------------|
| Debian 系统  | dpkg --install package_file |
| Red Hat 系统 | rpm -U package_file         |

当然此处的文件都是新版。

### 列出已安装的软件包列表

| 系统类型       | 命令行         |
|------------|-------------|
| Debian 系统  | dpkg --list |
| Red Hat 系统 | rpm -qa     |

由于内容过长，为了方便查看，可以将输出用管道交由`less`命令查看，完成后按`q`退出：

```shell
rpm -qa | less
---------------
dpkg --list | less
```

### 判断软件包是否安装

| 系统类型       | 命令行                        |
|------------|----------------------------|
| Debian 系统  | dpkg --status package_name |
| Red Hat 系统 | rpm -q package_name        |

```shell
dpkg --status emacs
dpkg-query: package 'emacs' is not installed and no information is available
Use dpkg --info (= dpkg-deb --info) to examine archive files.
```

### 显示已安装软件包的相关信息

在已知已安装软件包名称的情况下，可以使用如下命令查看描述信息

| 系统类型      | 命令行                         |
|-----------|-----------------------------|
| Debian 系统 | apt-cache show package_name |
| Red Hat   | yum info package_name       |

此处展示`vim`的安装信息：

```shell
apt-cache show vim | less
----------------------------
Package: vim
Architecture: amd64
Version: 2:8.1.2269-1ubuntu5
Priority: optional
Section: editors
Origin: Ubuntu
Maintainer: Ubuntu Developers <ubuntu-devel-discuss@lists.ubuntu.com>
Original-Maintainer: Debian Vim Maintainers <pkg-vim-maintainers@lists.alioth.debian.org>
Bugs: https://bugs.launchpad.net/ubuntu/+filebug
Installed-Size: 3038
Provides: editor
...

(END)
```

### 查看某具体文件由哪个安装包得到

| 系统类型      | 命令行                     |
|-----------|-------------------------|
| Debian 系统 | dpkg --search file_name |
| Red Hat   | rpm -qf file_name       |

同样的我们来检查一下`vim`：

```shell
[asuka@localhost ~]$ rpm -qf /usr/bin/vim
vim-enhanced-8.0.1763-10.el8.x86_64
```
