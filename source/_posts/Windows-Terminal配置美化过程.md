---
title: Windows Terminal配置美化过程
tags:
  - 技术
categories:
  - 工具
date: 2022-11-07 19:16:04
---

## 1. 预先准备

### 1.1 美化PowerShell

在ArchLinux上用的是Oh-My-Zsh。为了符合习惯，使用类似的Posh-Git和Oh-My-Posh，在PowerShell中执行以下的命令进行安装：

```powershell
Install-Module posh-git -Scope CurrentUser
Install-Module oh-my-posh -Scope CurrentUser
```

打开 PowerShell 的配置文件：

```powershell
notepad $PROFILE
```

添加以下内容，本人选择的主题是`Agnoster`，为了与Oh-My-Zsh的主题保持一致性。

```powershell
Import-Module posh-git
Import-Module oh-my-posh
Set-Theme Agnoster
```

特别注意字体，一定要使用支持Powerline的字体，否则会产生乱码。

## 1.2 PSReadLine的使用

`PSReadLine`是一个用于增强PowerShell命令行编辑体验的模块，其安装和使用说明请参考其开源项目主页。由于使用的复杂性，为了简单起见，只加入自动补全功能：

仍是在PowerShell的配置文件中输入，即可实现在zsh下的Menu补全。

```powershell
Set-PSReadlineKeyHandler -Key Tab -Function MenuComplete
```

## 2. 配置

### 2.1 Global Setting 配置

#### 2.1.1 对所有的Profile进行设置

不搞花里胡哨的背景图片。所有Profile均采用同样的风格。

```json
{
    "profiles": {
        "defaults": {
            "fontFace": "MesloLG NF",
            "fontSize": 14,
            "colorScheme": "Dracula",
            "acrylicOpacity": 0.8,
            "useAcrylic": true
        }
    }
}
```

#### 2.1.2 退出 Windows Terminal

```json
{ "command": "closeWindow", "keys": "alt+shift+q" }
```

### 2.2 Color Schemes 配置

这个主题配置主题颜色的JSON文件来源为开源项 iTerm2-Color-Schemes。因为本人在VsCode采用的主题也是Dracula，为了保持一致性，Windows Terminal的配色主题也是采用Dracula。

```json
{ "schemes": [{
     "name": "Dracula",
     "black": "#000000",
     "red": "#ff5555",
     "green": "#50fa7b",
     "yellow": "#f1fa8c",
     "blue": "#bd93f9",
     "purple": "#ff79c6",
     "cyan": "#8be9fd",
     "white": "#bbbbbb",
     "brightBlack": "#555555",
     "brightRed": "#ff5555",
     "brightGreen": "#50fa7b",
     "brightYellow": "#f1fa8c",
     "brightBlue": "#bd93f9",
     "brightPurple": "#ff79c6",
     "brightCyan": "#8be9fd",
     "brightWhite": "#ffffff",
     "background": "#1e1f29",
     "foreground": "#f8f8f2"
 }]
}
```

为了使所有的Profile能够使用这个配色主题，在Profile中的defaults添加如下信息：

```json
{
    "defaults": {
        "colorScheme": "Dracula"
    }
}
```

### 2.3 Key bindings

由于Linux系统使用ArchLinux i3，为了符合ArchLinux i3的习惯。此处的Modifier只会采用Alt，而且设置习惯将会根据i3的快捷键设置。

#### 2.3.1 Tab 管理命令配置

用于创建新的tab，以及默认tab的打开：

```json
[{ "command": "newTab", "keys": "alt+enter" },
{ "command": { "action": "newTab", "index": 0 }, "keys": "alt+shift+1" },
{ "command": { "action": "newTab", "index": 1 }, "keys": "alt+shift+2" },
{ "command": { "action": "newTab", "index": 2 }, "keys": "alt+shift+3" },
{ "command": { "action": "newTab", "index": 3 }, "keys": "alt+shift+4" },
{ "command": { "action": "newTab", "index": 4 }, "keys": "alt+shift+5" },
{ "command": { "action": "newTab", "index": 5 }, "keys": "alt+shift+6" },
{ "command": { "action": "newTab", "index": 6 }, "keys": "alt+shift+7" },
{ "command": { "action": "newTab", "index": 7 }, "keys": "alt+shift+8" },
{ "command": { "action": "newTab", "index": 8 }, "keys": "alt+shift+9" }]
```

用于创建一个当前tab的副本并打开:

```json
{ "command": "duplicateTab", "keys": "alt+shift+d" }
```

用于关闭当前的tab：

```json
{ "command": "closeTab", "keys": "alt+q" }
```

用于切换tab：

```json
[{ "command": { "action": "switchToTab", "index": 0 }, "keys": "alt+1" },
{ "command": { "action": "switchToTab", "index": 1 }, "keys": "alt+2" },
{ "command": { "action": "switchToTab", "index": 2 }, "keys": "alt+3" },
{ "command": { "action": "switchToTab", "index": 3 }, "keys": "alt+4" },
{ "command": { "action": "switchToTab", "index": 4 }, "keys": "alt+5" },
{ "command": { "action": "switchToTab", "index": 5 }, "keys": "alt+6" },
{ "command": { "action": "switchToTab", "index": 6 }, "keys": "alt+7" },
{ "command": { "action": "switchToTab", "index": 7 }, "keys": "alt+8" },
{ "command": { "action": "switchToTab", "index": 8 }, "keys": "alt+9" }]
```

#### 2.3.2 Pane 管理命令配置

关闭当前聚焦的pane。如果没有Pane，就会直接关闭掉当前的Tab。如果只有一个Tab打开，将会关闭窗口：

```json
{ "command": "closePane", "keys": "alt+w" }
```

改变当前聚焦的pane的尺寸：

```json
[{ "command": { "action": "resizePane", "direction": "down" }, "keys": "alt+shift+down" },
{ "command": { "action": "resizePane", "direction": "left" }, "keys": "alt+shift+left" },
{ "command": { "action": "resizePane", "direction": "right" }, "keys": "alt+shift+right" },
{ "command": { "action": "resizePane", "direction": "up" }, "keys": "alt+shift+up" }]
```

利用当前的profile划分pane：

```json
[{ "command": { "action": "splitPane", "split": "horizontal"}, "keys": "alt+h" },
{ "command": { "action": "splitPane", "split": "vertical"}, "keys": "alt+v" }]
```

利用其他的profile进行分割。由于`alt+index`以及`alt+shift+index`均用于了Tab的管理，此处不得不采用Modifier `ctrl`：

```json
[{ "command": { "action": "splitPane", "split": "auto", "index": 0 }, "keys": "ctrl+alt+1" },
{ "command": { "action": "splitPane", "split": "auto", "index": 1 }, "keys": "ctrl+alt+2" },
{ "command": { "action": "splitPane", "split": "auto", "index": 2 }, "keys": "ctrl+alt+3" },
{ "command": { "action": "splitPane", "split": "auto", "index": 3 }, "keys": "ctrl+alt+4" },
{ "command": { "action": "splitPane", "split": "auto", "index": 4 }, "keys": "ctrl+alt+5" },
{ "command": { "action": "splitPane", "split": "auto", "index": 5 }, "keys": "ctrl+alt+6" },
{ "command": { "action": "splitPane", "split": "auto", "index": 6 }, "keys": "ctrl+alt+7" },
{ "command": { "action": "splitPane", "split": "auto", "index": 7 }, "keys": "ctrl+alt+8" },
{ "command": { "action": "splitPane", "split": "auto", "index": 8 }, "keys": "ctrl+alt+9" }]
```

#### 2.3.3 字体大小重置

```json
{ "command": "resetFontSize", "keys": "alt+r" }
```

## 3. 配置文件全部

此处给出所有的配置文件，供参考。

```json
{
    //! Do not edit this
    "$schema": "https://aka.ms/terminal-profiles-schema",

    //! To set the default profile to PowerShell7
    "defaultProfile": "{574e775e-4f2a-5b96-ac1e-a2962a402336}",

    //! To define the profiles
    "profiles": {
        //! Settings to apply to all profiles
        "defaults": {
            "fontFace": "MesloLGS NF",
            "fontSize": 14, // 老年人了
            "colorScheme": "Dracula",
            "acrylicOpacity": 0.8,
            "useAcrylic": true
        },
        //! Profile objects
        "list": [
            //! PowerShell 7
            {
                "guid": "{574e775e-4f2a-5b96-ac1e-a2962a402336}",
                "tabTitle": "PowerShell 7",
                "hidden": false,
                "name": "PowerShell",
                "source": "Windows.Terminal.PowershellCore"

            },
            //! Windows PowerShell 5.1
            {
                "guid": "{61c54bbd-c2c6-5271-96e7-009a87ff44bf}",
                "name": "Windows PowerShell",
                "commandline": "powershell.exe",
                "hidden": false
            },
            //! Don't need at this time
            {
                "guid": "{b453ae62-4e3d-5e58-b989-0a998ec441b8}",
                "hidden": true,
                "name": "Azure Cloud Shell",
                "source": "Windows.Terminal.Azure"
            },
            //! CMD
            {
                // Make changes here to the cmd.exe profile.
                "guid": "{0caa0dad-35be-5f56-a8ff-afceeeaa6101}",
                "name": "命令提示符",
                "commandline": "cmd.exe",
                "hidden": false,
                "tabTitle": "cmd"
            }


        ]
    },

    //! To define the Dracula theme
    "schemes": [{
            "name": "Dracula",
            "black": "#000000",
            "red": "#ff5555",
            "green": "#50fa7b",
            "yellow": "#f1fa8c",
            "blue": "#bd93f9",
            "purple": "#ff79c6",
            "cyan": "#8be9fd",
            "white": "#bbbbbb",
            "brightBlack": "#555555",
            "brightRed": "#ff5555",
            "brightGreen": "#50fa7b",
            "brightYellow": "#f1fa8c",
            "brightBlue": "#bd93f9",
            "brightPurple": "#ff79c6",
            "brightCyan": "#8be9fd",
            "brightWhite": "#ffffff",
            "background": "#1e1f29",
            "foreground": "#f8f8f2"
        }

    ],

    "keybindings": [{
            "command": {
                "action": "copy",
                "singleLine": false
            },
            "keys": "ctrl+c"
        },

        {
            "command": "paste",
            "keys": "ctrl+v"
        },

        //! To open the search box
        {
            "command": "find",
            "keys": "alt+f"
        },

        //! To exit the Windows Terminal
        {
            "command": "closeWindow",
            "keys": "alt+shift+q"
        },

        //! To open the default tab
        {
            "command": "newTab",
            "keys": "alt+enter"
        },

        //! To duplicate the current tab and open it
        {
            "command": "duplicateTab",
            "keys": "alt+shift+d"
        },

        //* To open the tab by index
        {
            "command": {
                "action": "newTab",
                "index": 0
            },
            "keys": "alt+shift+1"
        },
        {
          "command": {
            "action": "newTab",
            "index": 1
          },
            "keys": "alt+shift+2"
        },
        {
            "command": {
                "action": "newTab",
                "index": 2
            },
            "keys": "alt+shift+3"
        },
        {
            "command": {
                "action": "newTab",
                "index": 3
            },
            "keys": "alt+shift+4"
        },
        {
            "command": {
                "action": "newTab",
                "index": 4
            },
            "keys": "alt+shift+5"
        },
        {
            "command": {
                "action": "newTab",
                "index": 5
            },
            "keys": "alt+shift+6"
        },
        {
            "command": {
                "action": "newTab",
                "index": 6
            },
            "keys": "alt+shift+7"
        },
        {
            "command": {
                "action": "newTab",
                "index": 7
            },
            "keys": "alt+shift+8"
        },
        {
            "command": {
                "action": "newTab",
                "index": 8
            },
            "keys": "alt+shift+9"
        },
        //* End

        //* To switch to a tab by index
        {
            "command": {
                "action": "switchToTab",
                "index": 0
            },
            "keys": "alt+1"
        },
        {
            "command": {
                "action": "switchToTab",
                "index": 1
            },
            "keys": "alt+2"
        },
        {
            "command": {
                "action": "switchToTab",
                "index": 2
            },
            "keys": "alt+3"
        },
        {
            "command": {
                "action": "switchToTab",
                "index": 3
            },
            "keys": "alt+4"
        },
        {
            "command": {
                "action": "switchToTab",
                "index": 4
            },
            "keys": "alt+5"
        },
        {
            "command": {
                "action": "switchToTab",
                "index": 5
            },
            "keys": "alt+6"
        },
        {
            "command": {
                "action": "switchToTab",
                "index": 6
            },
            "keys": "alt+7"
        },
        {
            "command": {
                "action": "switchToTab",
                "index": 7
            },
            "keys": "alt+8"
        },
        {
            "command": {
                "action": "switchToTab",
                "index": 8
            },
            "keys": "alt+9"
        },
        //* End

        //! To close the tab
        {
            "command": "closeTab",
            "keys": "alt+q"
        },

        //! To close focused pane
        {
            "command": "closePane",
            "keys": "alt+w"
        },

        //* To resize a pane
        {
            "command": {
                "action": "resizePane",
                "direction": "down"
            },
            "keys": "alt+shift+down"
        },
        {
            "command": {
                "action": "resizePane",
                "direction": "left"
            },
            "keys": "alt+shift+left"
        },
        {
            "command": {
                "action": "resizePane",
                "direction": "right"
            },
            "keys": "alt+shift+right"
        },
        {
            "command": {
                "action": "resizePane",
                "direction": "up"
            },
            "keys": "alt+shift+up"
        },
        //* End

        //* To split a pane within current profile
        {
            "command": {
                "action": "splitPane",
                "split": "horizontal"
            },
            "keys": "alt+h"
        },
        {
            "command": {
                "action": "splitPane",
                "split": "vertical"
            },
            "keys": "alt+v"
        },
        //* End

        //* To split a pane with another profile
        {
            "command": {
                "action": "splitPane",
                "split": "auto",
                "index": 0
            },
            "keys": "ctrl+alt+1"
        },
        {
            "command": {
                "action": "splitPane",
                "split": "auto",
                "index": 1
            },
            "keys": "ctrl+alt+2"
        },
        {
            "command": {
                "action": "splitPane",
                "split": "auto",
                "index": 2
            },
            "keys": "ctrl+alt+3"
        },
        {
            "command": {
                "action": "splitPane",
                "split": "auto",
                "index": 3
            },
            "keys": "ctrl+alt+4"
        },
        {
            "command": {
                "action": "splitPane",
                "split": "auto",
                "index": 4
            },
            "keys": "ctrl+alt+5"
        },
        {
            "command": {
                "action": "splitPane",
                "split": "auto",
                "index": 5
            },
            "keys": "ctrl+alt+6"
        },
        {
            "command": {
                "action": "splitPane",
                "split": "auto",
                "index": 6
            },
            "keys": "ctrl+alt+7"
        },
        {
            "command": {
                "action": "splitPane",
                "split": "auto",
                "index": 7
            },
            "keys": "ctrl+alt+8"
        },
        {
            "command": {
                "action": "splitPane",
                "split": "auto",
                "index": 8
            },
            "keys": "ctrl+alt+9"
        },
        //* End

        //! To reset font size
        {
            "command": "resetFontSize",
            "keys": "alt+r"
        }
    ]
}
```
