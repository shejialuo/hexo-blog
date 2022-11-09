---
title: Telegram代收QQ和微信消息
tags:
  - 技术
categories:
  - 工具
date: 2022-11-07 23:32:54
redirect_from:
  - /e4e0c5921655
---

## 1. 前言

在ArchLinux i3上想用QQ和微信别人交流，是一件极其困难的事情，腾讯开发的linux for qq总是经常的出现段错误，而且界面过于丑陋。同时，腾讯为UOS开发的WeChat仅仅通过Electron封装了网页端，聊天记录都不能保存。故尝试采用Telegram代收QQ和微信消息。首先本文感谢以下的开源项目：

+ [efb-qq-slave](https://github.com/ehForwarderBot/efb-qq-slave)
+ [efb-qq-plugin-go-cqhttp](https://github.com/ehForwarderBot/efb-qq-plugin-go-cqhttp)
+ [go-cqhttp](https://github.com/Mrs4s/go-cqhttp)
+ [efb-wechat-slave](https://github.com/ehForwarderBot/efb-wechat-slave)
+ [efb-telegram-master](https://github.com/ehForwarderBot/efb-telegram-master)
+ [enForwarderBot](https://github.com/ehForwarderBot/ehForwarderBot)

在阅读本教程以前，你应该首先阅读`efb-qq-slave`作者写的教程[安装并使用EFB：在Telegram收发QQ消息](https://milkice.me/2018/09/17/efb-how-to-send-and-receive-messages-from-qq-on-telegram/)。熟悉一些基本的概念。

## 2. 前提条件

在开始部署之前，需要完成以下的准备工作：

+ 墙外的一台VPS或者墙内的一台VPS加上科学上网工具
+ 一台电脑，Windows系统采用Windows Terminal和PowerShell7进行SSH连接

## 3. 环境搭建

由于我的服务器环境为Ubuntu 20.04 LTS，本文按照该操作系统进行描述。如果你采用的是其他发行版的Linux服务器，其过程并无本质区别。由于依赖或LTS版本的问题，你可能会遇到很多坑，如果你熟悉Docker，建议采取Docker部署服务，同时也为你的服务器中的其他服务提供了隔离。

### 3.1 enForwarderBot

我自己写了一个shell脚本进行安装，故此处展示函数，有需要请读者自取。

```sh
install_efb() {
  # To install the corresponding dependency
  sudo apt-get install libopus0 ffmpeg libmagic1 git libssl-dev

  # To install the efb
  pip3 install -U https://github.com/ehForwarderBot/ehForwarderBot
}
```

### 3.2 efb-telegram-master

```sh
install_etm() {
  # To install the efb-telegram-master
  pip3 install -U git+https://github.com/ehforwarderbot/efb-telegram-master
}
```

### 3.3 efb-qq-slave

```sh
install_eqs() {
  # To install efb-qq-slave
  pip3 install -U git+https://github.com/ehForwarderBot/efb-qq-slave
  # To install efb-qq-plugin-go-cqhttp
  pip3 install -U git+https://github.com/ehForwarderBot/efb-qq-plugin-go-cqhttp
}
```

### 3.4 efb-wechat-slave

```sh
install_ews() {
  # To install efb-wechat-slave
  pip3 install -U git+https://github.com/ehForwarderBot/efb-wechat-slave
}
```

### 3.5 中间件安装

此处有少数可选的中间件，用于加强功能。你可以选择安装也可以直接忽略。

```sh
install_middlewares() {
  # To install efb-link_preview-middleware
  pip3 install -U git+https://github.com/ehForwarderBot/efb-link_preview-middleware

  # To install efb-voice_recog-middleware
  pip3 install -U git+https://github.com/ehForwarderBot/efb-voice_recog-middleware

  # To install efb-search_msg-middleware
  pip3 install -U git+https://github.com/ehForwarderBot/efb-search_msg-middleware
}
```

## 4. 配置

EFB的工作机制是一个唯一的主端和多个从端进行通信。由于需要同时代收QQ和微信，故需要两个Telegram机器人，一个机器人用于接收QQ消息，一个机器人用于接收微信消息。

### 4.1 配置两个Telegram机器人

此处直接参考efb-qq-slave作者的博客的教程，你需要重复该操作两次。一个是为了接收QQ消息，一个是为了接收微信消息。

要创建一个新的Bot，要先向`@BotFather`发起会话。发送指令`/newbot`以启动向导。期间，你需要指定这个 Bot的名称与用户名（用户名必须以bot结尾）。完毕之后`@BotFather`会提供给你一个密钥（Token），妥善保存这个密钥。请注意，为保护您的隐私及信息安全，请不要向任何人提供你的Bot用户名及密钥，这可能导致聊天信息泄露等各种风险。

接下来还要对刚刚启用的 Bot 进行进一步的配置：允许 Bot 读取非指令信息、允许将 Bot 添加进群组、以及提供指令列表。

发送 `/setprivacy` 到 `@BotFather`，选择刚刚创建好的Bot用户名，然后选择 “Disable”.
发送 `/setjoingroups` 到 `@BotFather`，选择刚刚创建好的Bot用户名，然后选择 “Enable”.
发送 `/setcommands` 到 `@BotFather`，选择刚刚创建好的Bot用户名，然后发送如下内容：

```txt
help - Show commands list.
link - Link a remote chat to a group.
unlink_all - Unlink all remote chats from a group.
info - Display information of the current Telegram chat.
chat - Generate a chat head.
extra - Access additional features from Slave Channels.
update_info - Update info of linked Telegram group.
react - Send a reaction to a message, or show a list of reactors.
rm - Remove a message from its remote chat.
```

然后还需要获取你自己的 Telegram ID，ID 应显示为一串数字。获取你自己的 ID 有很多方式，你可以选择任意一种。下面介绍两种可能的方式。

1. Plus Messenger，如果你使用了Plus Messenger作为你的Telegram客户端，你可以直接打开你自己的资料页，在「自己」下面会显示你的ID。
2. 通过Bot查询，很多现存的Bot也提供了ID查询服务，直接向其发送特定的指令即可获得自己的数字ID。在这里介绍一些接触过的。

+ @get_id_bot 发送 /start
+ @XYMbot 发送 /whois
+ @mokubot 发送 /whoami
+ @GroupButler_Bot 发送 /id
+ @jackbot 发送 /me
+ @userinfobot 发送任意文字
+ @orzdigbot 发送 /user

留存你的Telegram ID以便后续使用。

### 4.2 配置QQ

如果需要配置QQ，你必须要首页阅读这两个项目主页的帮助文档：

+ [efb-qq-slave](https://github.com/ehForwarderBot/efb-qq-slave)
+ [efb-qq-plugin-go-cqhttp](https://github.com/ehForwarderBot/efb-qq-plugin-go-cqhttp)

然后你可以执行参考以下的脚本，你只需要修改`token`和`id`。

```sh
QQ_CONFIG=~/.ehforwarderbot/profiles/qq
TELEGRAM_MASTER="blueset.telegram"
QQ_SLAVE="milkice.qq"
configure_qq() {
  if [[  -e "${QQ_CONFIG}" ]]; then
    echo "The profile directory exists, exit"
    exit 1
  fi
  mkdir -p "${QQ_CONFIG}"
  mkdir -p "${QQ_CONFIG}/${TELEGRAM_MASTER}"
  mkdir -p "${QQ_CONFIG}/${QQ_SLAVE}"

  (
    echo "master_channel: ${TELEGRAM_MASTER}"
    echo "slave_channels:"
    echo "  - ${QQ_SLAVE}"
  ) >> "${QQ_CONFIG}/config.yaml"

  (
    echo 'token: "<your_token>"'
    echo "admins:"
    echo "  - <your_id>"
  ) >> "${QQ_CONFIG}/${TELEGRAM_MASTER}/config.ymal"

  (
    echo "Client: GoCQHttp"
    echo "GoCQHttp:"
    echo "  type: HTTP"
    echo "  access_token:"
    echo "  api_root: http://127.0.0.1:5700/"
    echo "  host: 127.0.0.1"
    echo "  port: 8000"
  ) >>  "${QQ_CONFIG}/${QQ_SLAVE}/config.ymal"

}
```

同时你需要运行QQ机器人，此处请参考仓库里面的教程。

### 4.3 配置微信

同样，你应该首先阅读下面仓库的说明文档：

+ [efb-wechat-slave](https://github.com/ehForwarderBot/efb-wechat-slave)

然后你可以执行参考以下的脚本，你只需要修改`token`和`id`。

```sh
WECHAT_WEB_CONFIG=~/.ehforwarderbot/profiles/wechat
TELEGRAM_MASTER="blueset.telegram"
WECHAT_SLAVE="blueset.wechat"
configure_wechat_web() {
  if [[  -e "${WECHAT_WEB_CONFIG}" ]]; then
    echo "The profile directory exists, exit"
    exit 1
  fi
  mkdir -p "${WECHAT_WEB_CONFIG}"
  mkdir -p "${WECHAT_WEB_CONFIG}/${TELEGRAM_MASTER}"
  mkdir -p "${WECHAT_WEB_CONFIG}/${WECHAT_SLAVE}"

  (
    echo "master_channel: ${TELEGRAM_MASTER}"
    echo "slave_channels:"
    echo "  - ${WECHAT_SLAVE}"
  ) >> "${WECHAT_WEB_CONFIG}/config.yaml"

  (
    echo 'token: "<your_token>"'
    echo "admins:"
    echo "  - <your_id>"
  ) >> "${WECHAT_WEB_CONFIG}/${TELEGRAM_MASTER}/config.ymal"

}
```

## 5. 运行

### 5.1 前台运行用于测试

对于QQ而言：

```sh
# 首先运行QQ机器人
./go-cqhttp
# 启动efb-qq-slave
ehforwarderbot --profile qq
```

对于微信而言：

```sh
ehforwarderbot --profile wechat
```

### 5.2 使用服务

创建后台服务，对于QQ而言，你可以创建两个服务，一个启动机器人，一个启动`efb-qq-slave`

对于微信而言，你只需要创建一个服务直接启动`efb-wechat-slave`。至于怎么创建服务，请读者自行查阅相关资料。
