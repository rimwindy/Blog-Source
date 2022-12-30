---
title: Arch Linux安装指南
tags: [Arch Linux, 安装指南]
date: 2019-11-11 11:41
updated: 2022-11-13 18:17
categories: 笔记本
---

{% note info Note %}

本指南已过时，建议参考最新 Wiki 或其他博主的整理。如：
* [ArchWiki - Installation guide](https://wiki.archlinux.org/title/Installation_guide_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87))
* [archlinux 简明指南](https://arch.icekylin.online/)

{% endnote %}

最近又换回Arch啦，听闻base软件包组已经被同名软件元包取缔（目前二者无法相互替代），所以安装过程又有变化了。鉴于自己也是经常需要找Wiki查看安装手册，也需要在安装后到处找配置相关的信息，甚感繁琐，故此写一篇比较完整的安装手记，方便日后使用。

> 因为内容比较多，打算先写一个大致的框架，一些简单的操作和细节补充就先跳过了，等后续有时间再慢慢填坑吧~

# *Preview*
![dolphin](/img/2019/11/11/0.png)

![lockscreen](/img/2019/11/11/1.png)

![neofetch](/img/2019/11/11/2.png)

# 安装前的准备

## 下载镜像
可以去官方提供的下载处进行下载：https://www.archlinux.org/download
目前提供的有BT种子和磁力链接，HTTP/HTTPS下载的话直接往下翻，找到中国的镜像站点进行下载就好了。下载完成后可以查看一下文件的散列值（另一种译名叫哈希）验证完整性，Windows上的工具也很多，如果你之前没有用过的话，也不必专门去下载了，可以使用自带的Power Shell来完成：
```bash
# Usage:
Get-FileHash <filepath> -Algorithm MD5
```
按照计算出的MD5值与官网给出的比对一下即可：
![MD5](/img/2019/11/11/3.png)

## 确定启动类型

> 因为不同的启动模式安装方法会有少许的不同，所以需要特别注意一下。现在大多数都是UEFI的，所以MBR就先鸽了，之后再补充吧￣へ￣

确定启动类型的方法有很多：
* 可以按下win + r，在弹出的运行框中输入msinfo32，回车后会打开系统信息，在里面有一个BIOS模式，看看是不是UEFI模式（近几年比较新的电脑一般都是UEFI）。
* 也可以右键win徽标键，选择磁盘管理，看看你的C盘最前面有没有一个EFI分区，有的话就说明是UEFI启动模式。

## 准备好硬盘空间
当然要准备好一块硬盘空间才能安装我们的系统了，可以使用win自带的磁盘管理，也可以使用第三方的比如分区助手之类的工具，大小看自己喜欢吧。

## 制作启动盘
准备好一个U盘，写入镜像可以使用rufus这款软件，官网在[这里](https://rufus.akeo.ie/)。
设备里选择你的U盘（记得备份数据，U盘会被清空），接着选择镜像，下面选择启动类型，UEFI就选UEFI，不是的话就选第一个。下面的可以默认：![rufus](/img/2019/11/11/4.png)

写入方式默认就好~
![rufus.png](/img/2019/11/11/5.png)

# 安装ing

## 设置启动顺序
这一步因为不同的电脑进入BIOS的按键不一样，所以需要大家自己百度一下。比如我的联想拯救者，BIOS键就是F2，只需要在开机时出现logo时快速点按BIOS键就可以了。进入BIOS后找到boot栏，找到你的U盘，将其调整为第一启动项（就是位置放在最上面），然后保存退出即可，不出意外的话，再次启动就可以看到arch的安装界面了，选择第一项即可进入到Arch的live安装环境。

## 配置系统

### 联网
Arch的安装是需要联网的，所以需要保证网络畅通。如果是自动获取ip的有线网络（比如把电脑的网线直接插到路由器上），应该什么都不用做，测试一下网络吧：
```bash
ping www.baidu.com
```
如果输出一串64 bytes from xxx (xxx): icmp_seq=xxx tti=xxx time=xxxms的东西就表示联网成功了。
如果用的是WiFi的话，可以使用
```bash
wifi-menu
```
接下来正常输入密码就可以使用了。

### 更新系统时间
```bash
timedatectl set-ntp true
# 正常情况下这条指令应该是没有输出的，所以可以使用下面的命令检查服务状态
timedatectl status
```

### 查看硬盘分区
这一步比较重要，建议大家一定要看清楚再格盘。
可以使用lsblk来查看硬盘状态（确定刚才分好的硬盘）：
```bash
lsblk
```
就像这样：
![lsblk](/img/2019/11/11/6.jpg)

### 格式化分区
当分区建立好了，这些分区都需要使用适当的文件系统进行格式化。举个例子，如果根分区在 /dev/sdX1 上并且会使用 ext4 文件系统，运行：
```bash
# 根据上一步查看到的硬盘信息，将sdX1替换成你想要安装系统的磁盘号
mkfs.ext4 /dev/sdX1
```

### 挂载分区
如果你是单独一个Arch系统的话，则还应该分出一个新的EFI分区，如果是Win + Arch双系统的话，则需要挂载你原来的EFI分区到Arch的EFI分区上。

执行以下命令将根分区挂载到/mnt：
```bash
# 将sdX1替换为之前创建的根分区
mount /dev/sdX1 /mnt

mkdir /mnt/boot
# sdX2是你的Win的EFI分区，一定要看清楚，一般大小在100M，200M左右。
mount /dev/sdX2 /mnt/boot
```
挂载完成后可以用lsblk命令查看有没有挂载成功。比如我的硬盘情况：
![mount](/img/2019/11/11/7.jpg)
sda是我的机械硬盘，sda1-3分别对应D、E、F分区，nvme0n1是我的固态硬盘，p1-p4几个分区分别对应我的EFI分区、MBR分区、Windows的C盘、Arch根分区。我没分swap分区，也没有单独分出home，大家按自己情况选择吧~（没有swap分区可能无法正常休眠）

### 选择软件镜像库
软件仓库是软件包存储的地方，通常我们所说的软件仓库指在线软件仓库，亦即用户从互联网获取软件的地方，选择一个国内的镜像源可以加速我们的下载过程。

> 接下来的安装过程会用到Vim，如果没有接触过的话可以先看看一些[教程](https://coolshell.cn/articles/5426.html)，掌握基本的操作。

用Vim打开/etc/pacman.d/mirrorlist：
```bash
# 提示：输入路径时可以用Tab键补全
vim /etc/pacman.d/mirrorlist
```
在normal模式下输入/可以进行查找，比如中科大的源，就可以查找ustc。normal模式下按下dd可以剪切光标下的行，按gg回到文件首，按p将行粘贴到文件最前面的位置（优先级最高）。完成后输入:wq保存退出即可。就像下面这样：
![source](/img/2019/11/11/8.jpg)

然后可以先刷新一下软件包数据库：
```bash
pacman -Syy
```

### 安装基本包
使用pacstrap安装基本系统，目前base包已经被替换了，所以一些软件包需要手动安装，通常情况下有以下几个：
* 一个软件元包base，包含基本系统所需的依赖
* 额外的软件包组比如base-devel，包含常用的开发工具
* 一个内核，大概有以下几种：
  * linux : 当前的稳定版本内核
  * linux-lts : 当前的长期支持版本内核
  * linux-hardened : 来自 https://github.com/anthraxx/linux-hardened 的安全强化内核
  * linux-zen : 来自 https://github.com/zen-kernel 的预载一定量优化的内核
* 大多数情况下，应该需要安装固件包 linux-firmware
* 一个文字编辑器，比如Vi、Vim、nano等
* 管理所用文件系统的用户工具，比如e2fsprogs、ntfs-3g，分别支持ext4和NTFS，如果还有其他需要，可以参考[官方文档](https://wiki.archlinux.org/index.php/File_systems)的说明自行安装
* 要像刚才一样联网的话，还需要这些：dhcpcd netctl iw dialog wpa_supplicant networkmanager

比如我的安装示例：
```bash
pacstrap /mnt base base-devel linux linux-firmware vi vim e2fsprogs ntfs-3g dhcpcd netctl iw dialog wpa_supplicant networkmanager
```
回车后就是等待安装完成了。

### 配置Fstab

> 这fstab是干嘛用的呢？（简单来说就是自动挂载）
>
> fstab文件可用于定义磁盘分区，各种其他块设备或远程文件系统应如何装入文件系统。每个文件系统在一个单独的行中描述。这些定义将在引导时动态地转换为系统挂载单元，并在系统管理器的配置重新加载时转换。 在启动需要挂载的服务之前，默认设置会自动fsck和挂载文件系统。例如，systemd会自动确保远程文件系统挂载 （如NFS或Samba）仅在网络设置完成后启动。因此，在/etc/fstab中指定的本地和远程文件系统挂载应该是开箱即用的。

生成自动挂载分区的fstab文件，执行以下命令：
```bash
genfstab -L /mnt >> /mnt/etc/fstab
```
可以输出一下生成的文件来检查是否正确，执行以下命令：
```bash
cat /mnt/etc/fstab
```

### Chroot

Chroot意为Change root，相当于把操纵权交给我们新安装（或已经存在）的Linux系统，执行了这步以后，我们的操作都相当于在磁盘上新装的系统中进行。如果以后我们的系统滚挂了，还需要用U盘启动，然后将根分区挂载到/mnt下，再使用chroot进系统修复，所以说U盘用完了不要扔哦~

Change root 到新安装的系统：
```bash
arch-chroot /mnt
```

### 设置时区
```bash
# 设置时区为上海并生成相关文件
# ln -s <源文件> <目标> 创建一个符号链接
ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime

# 设置时间标准为UTC，并调整时间漂移:
hwclock --systohc
```
顺带说一下，如果是双系统的话，Win和Arch的时间会不一样，这是因为Windows会把硬件时钟认为是localtime，而Linux会认为是UTC时间，所以会有8个小时的时差，解决方法可以改Win，也可以改Linux，这里以Win为例，修改使其将硬件时钟认为是UTC时间即可（在早于 Windows 7 的系统上发现过这样做会出现一些严重的问题： http://www.cl.cam.ac.uk/~mgk25/mswish/ut-rtc.html ）。

使用注册表修改，可以将下面的命令输入到Power Shell（管理员模式）中执行，也可以新建一个后缀为.reg的注册表文件，将命令粘进去，右键执行：
```bash
Reg add HKLM\SYSTEM\CurrentControlSet\Control\TimeZoneInformation /v RealTimeIsUniversal /t REG_DWORD /d 1
```

### 本地化
设置使用的语言选项：
```bash
vim /etc/locale.gen
```
/etc/locale.gen 是一个仅包含注释文档的文本文件。指定需要的本地化类型，去掉对应行前面的注释符号（＃）就可以啦，用Vim打开就可以了，建议只选择带UTF-8的选项，比如：
* zh_CN.UTF-8 UTF-8
* zh_HK.UTF-8 UTF-8
* zh_TW.UTF-8 UTF-8
* en_US.UTF-8 UTF-8

执行 locale-gen 以生成 locale 讯息：
```bash
locale-gen
```

创建 locale.conf 并提交本地化选项：
> 将系统 locale 设置为en_US.UTF-8，系统的 Log 就会用英文显示，这样更容易问题的判断和处理。用户可以设置自己的 locale。不推荐在此设置任何中文locale，可能会导致tty乱码。
```bash
echo LANG=en_US.UTF-8 > /etc/locale.conf
```

### 网络
#### 设置一个你喜欢的主机名吧：
```bash
# 将myhostname改为你想要设置的主机名
echo myhostname > /etc/hostname
```
#### 添加对应的信息到 hosts:
编辑/etc/hosts文件：
```bash
vim /etc/hosts
```

在文件末添加如下内容：
```bash
127.0.0.1	localhost
::1             localhost
127.0.1.1	myhostname.localdomain	myhostname
```

### Root密码
为你的root用户设置一个密码（虽然绝大多数时候是用不到的）：
```bash
passwd
```
回车之后直接输入密码就好了，屏幕上应该是啥都没有的，正常输入就完事了~

### 设置sudo
因为 root 用户的权力很大而且很危险，所以轻易不会用到它。

所以就有了 sudo(substitute user do) 使得系统管理员可以授权特定用户或用户组作为 root 或其他用户执行某些（或所有）命令，同时还能够对命令及其参数提供审核跟踪。

sudo 应该已经作为 base-devel 的一部分装上去了，如果没有的话也可以自己手动安装一下：
```bash
pacman -S sudo
```
sudo 的配置文件是 /etc/sudoers，不过有一个方便的指令visudo可以帮我们代理编辑它（就是先编辑一个临时文件，然后检查有没有错误， 一切 OK 后再覆盖）。
```bash
visudo
```
没错，其实就是用vi打开的，所以操作方式也和vi一样，找到下面这行，并将%wheel前面的注释符去掉，像下面这样：
```bash
## Uncomment to allow members of group wheel to execute any command
%wheel ALL=(ALL) ALL
```

#### 新建普通用户
配置完sudo后也可以顺带设置一下我们平常使用的普通用户：
```bash
# 自行替换username为你的用户名
useradd -m -G wheel username
```
相关参数的解释可以看我的[上一篇](https://www.akaneym.com/post/install-arch-on-windows/#%E6%B7%BB%E5%8A%A0%E6%96%B0%E7%9A%84%E6%99%AE%E9%80%9A%E7%94%A8%E6%88%B7)文章，这里就不再赘述了。
为你的新用户添加密码（这个密码是要经常用的）：
```bash
# 自行替换username为你的用户名
passwd username
```

### 安装Intel-ucode（非IntelCPU可以跳过此步骤）
```bash
pacman -S intel-ucode
```

### 安装启动器
启动器就是加载我们操作系统的程序，它是 BIOS 或 UEFI 启动的第一个程序。它负责使用正确的内核参数加载内核, 并根据配置文件加载初始化 RAM disk。

如果对其它的启动管理器有兴趣的话，可以去看 https://wiki.archlinux.org/index.php/Arch_boot_process#Boot_loader 。

这里以Linux常用的GRUB为例：

* 首先安装os-prober，它可以配合Grub检测已经存在的系统，自动设置启动选项。
```bash
pacman -S os-prober
```
* 安装grub与efibootmgr两个包：
```bash
pacman -S grub efibootmgr
```
* 部署grub:
```bash
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=grub
```
* 生成配置文件：
```bash
grub-mkconfig -o /boot/grub/grub.cfg
```
在生成配置文件的过程中，应该可以看到Windows的启动项，没看到的话需要进入Arch之后重新生成配置。

* 如果报 grub-probe: error: cannot find a GRUB drive for /dev/sdb1, check your device.map 的错误，并且sdb1这个地方是你的u盘，这是u盘uefi分区造成的错误，对安装没有影响，可以不用理会。
* 如果报 WARNING: Failed to connect to lvmetad. Falling back to device scanning 这样的错误，也可以不用理会，这是因为chroot中/run是不可用的。

如果你不放心的话，也可以检查一下刚才生成的grub.cfg配置文件中有没有Windows的启动项：
```bash
vim /boot/grub/grub.cfg
```
一般在文件的末尾处可以找到，如果没有找到的话需要回头看一下哪里做错了，实在不行的话可以参考[这里](https://wiki.archlinux.org/index.php/GRUB/Tips_and_tricks#Combining_the_use_of_UUIDs_and_basic_scripting)编辑配置文件手动添加引导的分区入口。

### 重启
先退出chroot:
```bash
exit
```
现在基本的安装已经结束了，可以尝试重启一下试试能不能正常启动，如果一切顺利的话就可以再次登陆安装我们的桌面环境了，再次登陆的时候可以使用新建的普通用户。

# 安装后的配置

## 显卡驱动
建议先安装集显的驱动，如果有需要的话再装独显的。

* 如果你是Intel的集成显卡，可以装这个：
```bash
sudo pacman -S xf86-video-intel
```
* 如果是ATI/AMD的话，可以参照[官方文档](https://wiki.archlinux.org/index.php/ATI_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87))的说明安装相应的驱动（建议选闭源的驱动，性能相对来说会更好），我这里就懒得写了（逃

* 一种Intel + Nvidia显卡的解决方案：
在桌面环境安装完成后再尝试此选项，本来这部分应该放到后面的，但既然说到了显卡驱动，也就一并写上了~

```bash
# nvidia: 英伟达闭源驱动。如果使用自定义内核，那就安装nvidia-dkms
# bbswitch: 切换使用的节能工具。如果使用自定义内核，那就安装bbswitch-dkms
sudo pacman -S nvidia bbswitch

# 托盘程序（可视化切换及设置）。会自动安装核心程序
# 如果配置有archlinuxcn源，那么同样可以使用pacman来安装
# 如果使用KDE桌面，另有optimus-manager-qt-kde可供选择
yay -S optimus-manager-qt
```
安装完成后重启，应该就可以看到panel栏中的manager了，右键点击它，选择设置—Optimus，Switching method选择Bbswitch，确定保存。之后切换显卡就可以用右键方便地选择了。

更多详细内容可以去看原作者的README文档，项目地址[这儿](https://github.com/Askannz/optimus-manager)。

## Xorg
接下来安装桌面环境需要的基础包xorg
```bash
sudo pacman -S xorg
```

## 安装桌面环境
Linux下可选的桌面环境有很多，具体的可以去[官方文档](https://wiki.archlinux.org/index.php/Desktop_environment_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87))查看，这里就以常见的Gnome和KDE为例吧：
* GNOME
```bash
# 想要 GNOME 全家桶的话带上 gnome-extras
sudo pacman -S gnome
```

* KDE
```bash
# 直接安装软件包组（包含了很多软件包）即可
sudo pacman -S plasma kde-applications
```

## 安装桌面管理器
安装好了桌面环境包以后，我们需要安装一个图形化的桌面管理器来帮助我们登录并且选择我们使用的桌面环境，KDE建议配合sddm，GNOME建议用gdm。这里以sddm为例：

### 安装sddm
```bash
sudo pacman -S sddm
```

### 设置开机启动sddm服务
Arch用于管理系统服务的命令为systemctl，使用方法也非常简单，大致如下：
```bash
sudo systemctl start   服务名 （启动一项服务）
sudo systemctl stop    服务名 （停止一项服务）
sudo systemctl enable  服务名 （开机启动一项服务）
sudo systemctl disable 服务名 （取消开机启动一项服务）
```

所以这里我们就执行下面命令来设置开机启动sddm：
```bash
sudo systemctl enable sddm
```

## 配置网络
由于我们之前使用的一直都是netctl这个自带的网络服务，而桌面环境使用的是NetworkManager这个网络服务，所以我们需要禁用netctl并启用NetworkManager：
```bash
sudo systemctl disable netctl
sudo systemctl enable NetworkManager
```

## 中文字体的安装
用pacman就可以方便的安装啦，选择你自己喜欢的字体吧：
* Google Noto Fonts 系列： noto-fonts noto-fonts-cjk noto-fonts-emoji

* 思源黑体：adobe-source-han-sans-otc-fonts

* 文泉驿：wqy-microhei wqy-zenhei

更多字体可以在[这里](https://wiki.archlinux.org/index.php/Fonts_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87))找到。

现在就可以重启你的计算机了，再次登陆的时候应该就可以看到你安装好的桌面环境了。

# 其他配置
其实到这里所有的安装都已经完成了，接下来主要是一些个人常用的软件、配置项的记录（不定时更新），方便以后重装的时候配置，大家也可以酌情参考~

## ZSH
具体内容[上一篇](https://www.akaneym.com/post/install-arch-on-windows/#%E5%AE%89%E8%A3%85%E9%85%8D%E7%BD%AEzsh)有写，这里不再赘述。

## 常用的命令行工具
```bash
sudo pacman -S neofetch git tree curl wget
```

## ffmpeg
```bash
sudo pacman -S ffmpeg
```

## 添加ArchLinuxCN
```bash
sudo vim /etc/pacman.conf
```
在文件末尾添加以下内容：
```bash
[archlinuxcn]
Server = https://mirrors.ustc.edu.cn/archlinuxcn/$arch
```
在安装archlinuxcn-keyring时可能会报本地秘钥无法签署的错误，这是因为pacman上游更新了密钥环的格式，这使得本地的主密钥无法签署其他密钥。解决方法如下：
```bash
# 以root身份运行
su
pacman -Syu haveged
systemctl start haveged
systemctl enable haveged

rm -fr /etc/pacman.d/gnupg
pacman-key --init
pacman-key --populate archlinux
pacman-key --populate archlinuxcn
```
完成后再次刷新一下：
```bash
sudo pacman -Syu
```

## yay
之前一个好用的AUR助手yaourt已经停止更新了，现在建议使用yay代替它：
```bash
git clone https://aur.archlinux.org/yay.git
cd yay
makepkg -si
```
安装完成后就可以使用yay -S xxx来下载软件了（注意前面不要加sudo）。
更新软件仓库：
```bash
yay -Syu
```

## Chrome
```bash
yay -S google-chrome
```

## VS Code
```bash
yay -S visual-studio-code-bin
```
> Visual Studio Code使用DBus传递菜单，如果全局菜单失效，可以尝试安装 libdbusmenu-glib

## 视频播放器
这里推荐使用VLC和MPV：
```bash
sudo pacman -S vlc mpv
```

## 截图软件
```bash
yay -S shutter
```

## latte-dock
```bash
sudo pacman -S latte-dock
```

## shadowsocks
命令行客户端：
```bash
sudo pacman -S shadowsocks
```

Usage:
* 使用ss-local 命令启动客户端
  启动客户端时使用/etc/shadowsocks/config.json配置文件:
  ```bash
  sslocal -c /etc/shadowsocks/config.json
  ```
* Shadowsocks的systemd服务可在/etc/shadowsocks/里调用不同的conf-file.json

  例如，在/etc/shadowsocks/中创建了foo.json配置文件，那么执行以下语句就可以调用该配置：

  * 启动shadowsocks：
  ```bash
  systemctl start shadowsocks@foo
  ```
  * 开机自启动shadowsocks：
  ```bash
  systemctl enable shadowsocks@foo
  ```

图形客户端：
```bash
sudo pacman -S shadowsocks-qt5
```

## 网易云音乐
```bash
yay -S netease-cloud-music
```

## 输入法
可用的有Ibus和Fcitx，这里以Fcitx为例：
```bash
sudo pacman -S fcitx fcitx-im
```
然后在~/.xprofile文件（没有就新建一个）中加入以下几行：
```bash
export GTK_IM_MODULE=fcitx
export QT_IM_MODULE=fcitx
export XMODIFIERS="@im=fcitx"
```
重启后应该就能使用输入法了~

## blender
```bash
sudo pacman -S blender
```

## steam
```bash
sudo vim /etc/pacman.conf
# 取消下面注释
[multilib]
Include = /etc/pacman.d/mirrorlist

# 安装32位驱动
sudo pacman -S lib32-nvidia-utils lib32-mesa-libgl lib32-mesa

# 安装steam和必需的运行组件
sudo pacman -S steam steam-native-runtime
```

## Minecraft
```bash
yay -S minecraft-launcher
```

# E.N.D
至此Arch Linux已经全部安装完成了，赶紧开始你的Arch之旅吧~

*Just Enjoy It*

# 参考文章
[Installation guide](https://wiki.archlinux.org/index.php/Installation_guide_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87))
[Arch Wiki](https://wiki.archlinux.org/)
[以官方Wiki的方式安装ArchLinux](https://www.viseator.com/2017/05/17/arch_install/)
[给 GNU/Linux 萌新的 Arch Linux 安装指南 rev.B](https://blog.yoitsu.moe/arch-linux/installing_arch_linux_for_complete_newbies.html)
[GnuPG-2.1 与 pacman 密钥环](https://www.archlinuxcn.org/gnupg-2-1-and-the-pacman-keyring/)