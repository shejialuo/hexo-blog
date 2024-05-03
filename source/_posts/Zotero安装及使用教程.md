---
title: Zotero安装及使用教程
tags:
  - 学术
categories:
  - 工具
date: 2021-04-22 17:42:41
excerpt: Zotero安装及使用教程
---

## 1. 安装

参考[官网教程](https://www.zotero.org/support/installation)。

在ArchLinux直接通过包管理器安装Zotero：

```sh
sudo pacman -S zotero
```

同时，安装完软件后，需要在浏览器安装插件[Zotero Connector](https://github.com/zotero/zotero-connectors)。

## 2. 基本概念定义

此部分主要阐述Zotero最重要的三个概念。

### 2.1 Item

在Zotero中存在着许多概念，这些概念在本质上都是基于Item概念的基础上定义的。官方文档给出的Item的定义如下所示：

> Every item contains different metadata, depending on what type it is. Items can be everything from books, articles, and reports to web pages, artwork, films, letters, manuscripts, sound recordings bills, cases, or statutes, among many others.

从官方文档的定义可以看出，Item就是一个结构体变量，包含了不同的元数据，如下图所示。

```c++
struct Item {
    ...
};
```

![Zotero的Item组成结构示意图](https://i.loli.net/2021/04/24/Kcs74rOiVJEl9ux.png)

### 2.2 Collection

音乐播放器的播放列表通常可以创建多个音乐列表，每个音乐列表可以包含不同的音乐。Collection的作用正如同播放列表一样，用于组织Item，如下图所示。

![Collection结构示意图](https://i.loli.net/2021/04/24/EhzfvaB5KnsNjAg.png)

### 2.3 Tag

Tag的含义很好理解，就是标签，另一种组织Item的方式。

## 3. 添加Item

本部分阐述添加Item的方法。

### 3.1 通过浏览器

首先需要在浏览器安装插件Zotero Connector。

假设找到了一篇文献[Microservices: Yesterday, Today, and Tomorrow](https://i.loli.net/2021/04/22/mVPRFHW8vpodecK.png)。打开网页后，使用插件直接添加Item，如下图所示。

![通过浏览器添加Item示意图](https://i.loli.net/2021/04/22/mVPRFHW8vpodecK.png)

同时Zotero还支持添加通用的网页和PDF文件作为Item，此处不赘述，还有一种情况就是当Zotero遇到某些网页含有多个Item的信息后，会弹出复选框进行选择，如下图所示。

![包含多个Item示意图](https://i.loli.net/2021/04/24/HEr2kLyN7QY1iUm.png)

### 3.2 通过标识符

Zotero同时支持通过标识符直接添加Item。可以添加以下的标识符：

+ [ISBN](https://en.wikipedia.org/wiki/International_Standard_Book_Number)
+ [DOI](https://en.wikipedia.org/wiki/Digital_object_identifier)
+ [PubMed ID](https://en.wikipedia.org/wiki/PubMed)
+ [arXiv ID](https://en.wikipedia.org/wiki/ArXiv)

[通过标识符添加Item](https://i.loli.net/2021/04/24/RY6WGgAEfxLMwmH.png)

## 4. 添加文件

文件可以作为一个单独的Item，也可以作为已经存在的Item的子Item。例如当存储一个PDF文件至Zotero时，Zotero会尝试获取该PDF文件的元数据并自动创建其父Item。如果无法识别其元数据，该PDF文件将会作为一个单独的Item存在。

Zotero的文件可以作为两种方式管理：

+ 实际文件：存储在[Zotero data directory](https://www.zotero.org/support/zotero_data)。
+ 链接文件：存储文件在磁盘中的位置。

### 4.1 通过浏览器添加

当Zotero通过浏览器添加Item时，Zotero Connector会自动识别与该Item相关的网页快照和PDF文件，保存的网页快照和PDF文件将会作为实际文件保存到Zotero data directory中。

### 4.2 通过Zotero软件添加

直接拖拽到相应的Item中即可，默认将会直接复制原文件至Zotero data directory中，如果需要使用链接文件，在拖拽的使用，按住如下按键：

+ Windows/Linux：Ctrl+Shift
+ macOS: Cmd+Option

也可以直接通过按钮添加文件，如下图所示：

![通过按钮添加文件](https://i.loli.net/2021/04/24/LOdG83VIYiNrZT6.png)

## 5. RSS订阅

Zotero也可以添加RSS订阅：

+ URL
+ OPML

## 6. 管理Item

直接通过按钮添加Collection，如下图所示。至于如何添加/移除Item到Collection、重命名Collection以及删除Collection的操作此处不赘述。

通过按钮添加Collection
通过按钮添加Collection

在Zotero中有几个特殊的Collection：

+ `My Publications`：顾名思义。
+ `Duplicate Items`：多余的Item，如何对其操作请参考官方文档。
+ `Unfiled Items`：包含没在任何一个Collection的Items。（无家可归的幽魂）
+ `Trash`：顾名思义。

Tag的操作也比较简单，此处不赘述，可参考官网文档。

## 7. 同步与备份

### 7.1 同步

Zotero的同步分为了两个类型：

+ 数据同步：可以直接使用Zotero提供的同步服务，免费且方便，创建一个Zotero的账户即可。
+ 文件同步

文件同步相对来说要复杂一下，首先是实际文件的同步。Zotero提供了对实际文件同步的功能，这一部分采取的是订阅制，免费用户可用的同步空间是300M。

当然也可以使用替代方案，通过WebDAV实现同步。目前国内比较好用的WebDAV也就只有坚果云了，目前使用坚果云实现同步的教程较多，此处不赘述。

除了对实际文件的同步以外，也可以进行对链接文件的同步，其本质的思路是在不同的主机上保存不同的文件链接。

### 7.2 备份

在Zotero data directory中最重要的文件是`zotero.sqlite`，同时也包含了目录`storage`，该目录主要是附件内容。

备份只需要对Zotero data directory进行备份即可，至于其他的备份方法，参考官方文档。

## 8. 可供参考资料

本教程几乎没有对细节的问题进行阐述，例如如何使用坚果云及其他国外支持WebDAV的网盘实现实际文件的同步、标签的具体使用、参考文献的导出和第三方插件的使用等问题。本教程着重点在于给读者对该软件有一个大方向的认识。因此在本教程的最后列出官网以及其他教程的参考资料供读者解决某些实际问题。（没有必要重复造轮子）

+ [Zotero官网参考资料](https://www.zotero.org/support/)
+ [坚果云使用 Zotero 配置过程详解](https://help.jianguoyun.com/?p=4190)
+ [Windows Word使用Zotero插入参考文献](https://www.zotero.org/support/)
+ [Zotero教程入门六篇](https://www.douban.com/group/topic/79928493/)
+ [知识管理软件Zotero的使用](https://zhuanlan.zhihu.com/p/26765443)

## 9. 个人使用感受

实际上，Zotero也有许多的插件，个人认为没有必要安装。例如使用MarkDown做笔记这个功能，为何不使用专门的MarkDown编辑器写然后添加文件到Item上呢？

Keep it simple and doing one thing well.
