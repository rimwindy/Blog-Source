---
title: WSL安装记
tags: [WSL, Arch Linux]
date: 2019-10-21 18:59
categories: 笔记本
thumbnail: /img/2019/10/21/0.jpg
---
# 缘起
Windows一直以来的一大痛点就是没有一个好用的命令行环境。此前虽然一直有听说 WSL ，但当时我还是用的多系统，所以并没有太多关注。在经过Windows -> Ubuntu -> Arch Linux -> 黑苹果 -> Windows的轮回后，我终于又用回了Windows 10(真香)。而作为 Linux 最好的发行版(雾)，怎么能没有一个好用的命令行环境呢？！虽然一直有折腾一下的想法，但~~平时课实在是太多了啊~~(其实是我太懒了(o´ω`o)ﾉ)于是就这样一直拖到了现在，今天正好有两节实验课比较水，完成任务后就开始了捣鼓，顺便~~记录一下过程~~(氵博客)。

## 关于WSL
Windows Subsystem for Linux（简称WSL）是一个在Windows 10上能够运行原生Linux二进制可执行文件（ELF格式）的兼容层。相较于虚拟机更为轻巧方便，安装也比较简单，开启WSL后在微软商店中直接搜索即可安装。

# 安装前的配置
## 启用WSL
控制面板->程序和功能->启用或关闭 window 功能->勾选“适用于 Linux 的 Windows 子系统”，之后重启系统。
![启用WSL](/img/2019/10/21/1.png)

## 打开开发者模式
设置->更新与安全->开发者选项->开发者模式
![选择开发者模式](/img/2019/10/21/2.png)

## 安装所需要的发行版
在微软商店搜索你需要安装的发行版，点击安装即可。我这里用的是Arch Linux，官方商店中并没有提供镜像，有以下两种方法解决：
* 使用非官方的安装包

  arch可以在github找到非官方的安装包，安装过程非常简单，下载完成后运行安装程序即可，仓库地址在[这儿](https://github.com/yuk7/ArchWSL)
  ![Arch的安装文件](/img/2019/10/21/3.png)

* 安装微软商店中的Ubuntu，按照Arch Wiki上的操作将内核换成Arch

  这里步骤比较多，我也并没有用这种方法，所以就懒得写了，地址我放在[这儿](https://wiki.archlinux.org/index.php/Install_on_WSL_(简体中文))了，大家有兴趣的话自己去看吧（逃~

# 安装完成后的配置

> 第一次安装程序时可能会报错，提示没有合法的数字签名。运行以下两条命令即可：
```bash
# 初始化 pacman 秘钥
pacman-key --init

# 验证主密钥
pacman-key --populate
```

## 安装VIM
```bash
pacman -S vim
```

## 配置国内的 mirrorlist 源
```bash
vim /etc/pacman.d/mirrorlist
```
这里以清华的源为例，删掉开头的注释符"#"使其生效（之前在实体机上安装的时候这些源都是未被注释的，即都生效，优先选择最上面的源，所以只需要将你需要的源放到第一个就行了，WSL这边似乎有些不一样，如果有知道的大佬可以解答一下~）
![更换源](/img/2019/10/21/4.png)

保存关闭后使用下面的命令刷新软件列表并更新软件
```bash
pacman -Syu
```

## 添加新的普通用户
> 注意：我之后的操作步骤都是在root身份下完成的，貌似windows terminal在非root环境下会有bug，反正我是用不了Tab补全，Ctrl +C清屏等功能的，不知道后续会不会修复，大家酌情选择吧~

使用命令：
```bash
useradd -m -G wheel username
```
username 替换为你的用户名

各参数的含义：

-m：在创建时同时在/home目录下创建一个与用户名同名的文件夹，这个目录就是你的家目录。这个神奇的目录将会用于存放你所有的个人资料、配置文件等所有跟系统本身无关的资料。这种设定带来了诸多优点：
* 只要家目录不变，你重装系统后只需要重新安装一下软件包（它们一般不存放在家目录），然后所有的配置都会从家目录中读取，完全不用重新设置软件。
* 你可以在家目录不变的情况下更换你的发行版而不用重新配置你的环境。
* 切换用户后所有的设置会从新的用户的家目录中读取，将不同用户的资料与软件设置等完全隔离。
* 有些著名的配置文件比如vim的配置文件~/.vimrc，只要根据自己的使用习惯配置一次， 在另一个Linux系统下（例如你的服务器）把这个文件复制到家目录下，就可以完全恢复你的配置。

-G wheel：-G代表把用户加入一个组，wheel就是组名，加入这个组是为了方便使用sudo命令。

### 为root和新用户设置密码
```bash
# root
passwd root

# other user
passwd username
```

### 配置sudo
```bash
visudo
```
找到 # %wheel ALL=(ALL)ALL 这一行，取消前面的注释符“#”。

这里的%wheel就是代表wheel组，意味着wheel组中的所有用户都可以使用sudo命令。

> root切换到普通用户可以使用 su - username 命令

## 安装配置zsh
虽然一般情况下默认的bash已经够用了，但是zsh更为强大高效，如果没有用过的话强烈推荐体验一下~
### 安装zsh
```bash
# 切换到用户文件夹
cd ~

# 安装zsh及必要的软件
pacman -S wget curl git
pacman -S zsh

# 下载oh-my-zsh,用于美化zsh，不注重这些的话可以不装
wget https://github.com/robbyrussell/oh-my-zsh/raw/master/tools/install.sh

# 赋予安装脚本可执行权限
chmod +x install.sh

./install.sh

# 更改系统默认的shell
chsh -s /bin/zsh
```
### 主题
首先了解几个重要的文件：

* zsh的配置文件 ： ~/.zshrc
* 主题的存放路径 ： ~/.oh-my-zsh/themes
* 插件的存放路径： ~/.oh-my-zsh/plugins

更改主题只需要在.zshrc文件中更改即可:
![更改主题](/img/2019/10/21/5.png)
更改完成后要使其立即生效只需执行：
```bash
source ~/.zshrc
```
配色根据自己用的terminal来改吧，我这里用的是windows terminal，配置文件有需要的话可以参考：
```json
{
    "$schema": "https://aka.ms/terminal-profiles-schema",

    "defaultProfile": "{61c54bbd-c2c6-5271-96e7-009a87ff44bf}",

    "profiles":
    [
        {
            "guid": "{61c54bbd-c2c6-5271-96e7-009a87ff44bf}",
            "name": "Windows PowerShell",
            "acrylicOpacity": 0.85,
            "useAcrylic": true,
            "cursorColor": "#777777",
            "closeOnExit": true,
            "colorScheme": "Night Owlish Light",
            "fontFace": "Monaco",
            "fontSize": 14,
            "commandline": "powershell.exe",
            "cursorShape": "filledBox",
            "scrollbarState": "hidden",
            "hidden": false
        },
        {
            "guid": "{0caa0dad-35be-5f56-a8ff-afceeeaa6101}",
            "name": "cmd",
            "acrylicOpacity": 0.85,
            "useAcrylic": true,
            "cursorColor": "#000000",
            "closeOnExit": true,
            "colorScheme": "Night Owlish Light",
            "commandline": "cmd.exe",
            "fontFace": "Monaco",
            "fontSize": 14,
            "cursorShape": "filledBox",
            "scrollbarState": "hidden",
            "hidden": false
        },
        {
            "guid": "{b453ae62-4e3d-5e58-b989-0a998ec441b8}",
            "hidden": false,
            "name": "Azure Cloud Shell",
            "source": "Windows.Terminal.Azure",
            "acrylicOpacity": 0.85,
            "useAcrylic": true,
            "cursorColor": "#000000",
            "closeOnExit": true,
            "colorScheme": "Night Owlish Light",
            "fontFace": "Monaco",
            "fontSize": 14,
            "cursorShape": "filledBox",
            "scrollbarState": "hidden"
        },
        {
            "guid": "{a5a97cb8-8961-5535-816d-772efe0c6a3f}",
            "hidden": false,
            "name": "Arch",
            "source": "Windows.Terminal.Wsl",
            "acrylicOpacity": 0.85,
            "useAcrylic": true,
            "cursorColor": "#777777",
            "closeOnExit": true,
            "colorScheme": "Night Owlish Light",
            "fontFace": "Monaco",
            "fontSize": 14,
            "icon": "E://Arch//archlinux.png", // 换上你自己图标的路径
            "scrollbarState": "hidden"
        }
    ],

    "schemes": [
        {
            "background": "#FFFFFF",
                "black": "#011627",
                "blue": "#4876D6",
                "brightBlack": "#7A8181",
                "brightBlue": "#5CA7E4",
                "brightCyan": "#00C990",
                "brightGreen": "#49D0C5",
                "brightPurple": "#697098",
                "brightRed": "#F76E6E",
                "brightWhite": "#989FB1",
                "brightYellow": "#DAC26B",
                "cyan": "#08916A",
                "foreground": "#403F53",
                "green": "#2AA298",
                "name": "Night Owlish Light",
                "purple": "#403F53",
                "red": "#D3423E",
                "white": "#7A8181",
                "yellow": "#DAAA01"
        }
    ],

    "keybindings": []
}
```
更多配色可以去[这里](https://iterm2colorschemes.com)找，更多关于Windows Terminal的配置项可以阅读[官方文档](https://github.com/microsoft/terminal/blob/master/doc/cascadia/SettingsSchema.md)

### 插件
下面介绍几个我正在用的zsh插件：
#### zsh-autosuggestions
自动补全插件，作用是记录下你曾经输入过的命令，之后再次输入时就会有类似IDE自动补全的效果，非常好用，安装方法可以直接用git clone：
```bash
git clone https://github.com/zsh-users/zsh-autosuggestions $ZSH_CUSTOM/plugins/zsh-autosuggestions
```
#### zsh-syntax-highlighting
语法高亮插件，能自动识别输入的命令是否有效，有效会显示成绿色，无效为红色：
```bash
git clone https://github.com/zsh-users/zsh-syntax-highlighting.git $ZSH_CUSTOM/plugins/zsh-syntax-highlighting
```
安装完成后修改一下.zshrc文件，使插件生效：
![应用插件](/img/2019/10/21/6.png)

同样的，保存退出后使配置文件生效：
```bash
source ~/.zshrc
```

***
# E.N.D
到这里差不多基本的配置就完成了，今天就先写到这儿吧，有什么问题欢迎留言，最后放一张成果图吧~
![Arch on Windows](/img/2019/10/21/7.png)

### 参考文章
[ArchLinux安装后的必须配置与图形界面安装教程](https://www.viseator.com/2017/05/19/arch_setup/)
[win10 arch子系统 docker](https://www.jianshu.com/p/a298e1c7b043)