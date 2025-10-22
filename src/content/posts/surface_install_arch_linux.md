---
title: Surface Pro 5 安装 Arch Linux 
published: 2025-10-22
tags: [Linux, Surface]
draft: false
---

---
# 安装前的准备工作
### 0. 下载 ISO 镜像文件

从 Arch Linux 官方网站下载最新的 ISO 镜像文件。
> [Arch Linux Downloads](https://archlinux.org/download/)

### 1. 刻录启动 U 盘

准备一个至少 2G 的 U 盘  

**在 Windows 上:**  

推荐使用 [Ventoy](https://www.ventoy.net/) 或 [Rufus](https://rufus.ie/)使用 Rufus 时，选择下载的 ISO 文件，并以 `DD` 模式写入  

**在 macOS/Linux 上:**

首先，找到你的 U 盘设备名（例如 `diskX` 或 `sdX`）。

```bash
# 在 macOS 上
diskutil list
# 在 Linux 上
lsblk
```
然后执行 `dd` 命令进行刻录  
**请务必将 `/dev/sdX` 替换为你的 U 盘设备名**  
**将 `/Users/lyq/Downloads/archlinux-2025.10.01-x86_64.iso` 替换为你下载到 ISO 文件的实际路径**  


```bash
# 示例命令
sudo dd if=/Users/lyq/Downloads/archlinux-2025.10.01-x86_64.iso of=/dev/sdX bs=4M status=progress oflag=sync
```

### 2. 设置 UEFI

1.  插入启动 U 盘
2.  将 Surface 关机  
3.  然后长按电源键和音量加键进入 UEFI  
4.  在 `Security` 选项卡中，将 `Secure Boot` 设置为 `Disabled`  
5.  在 `Boot configuration` 选项卡中，将 `USB Storage` 设置为第一启动项  
6.  在 `Boot configuration` 选项卡中，将 `Enable Boot from USB devices` 设置为 `On`  
7.  选择 `Exit` -> `Restart Now` 保存并重启  

不出意外你应该已经成功进入 Arch Linux 的安装界面

# 基础安装

### 0. 连接网络

```bash
# 启动 iwd 服务
systemctl start iwd
# 进入交互式命令行
iwctl
# 列出无线设备 (通常是 wlan0)
device list
# 扫描网络
station wlan0 scan
# 列出可用网络
station wlan0 get-networks
#进行连接 输入密码即可
station wlan0 connect YOUR-WIRELESS-NAME 
# 确认连接状态
station wlan0 show
# 退出 iwctl
exit
# 测试网络连接
ping archlinux.org
```

---
### 💡 可选：通过 SSH 远程连接安装（推荐）

如果需要使用其他设备通过 **SSH 连接**到 Arch Linux 安装环境（`archiso`）进行操作，以方便**复制粘贴命令**，请执行以下步骤。

**前提：** 确保 Surfacee 和用于 SSH 连接的设备处于同一局域网内。

#### 1. 设置 Root 密码

在 `archiso` 环境中，为 `root` 用户设置一个登录密码：

```bash
passwd
```

系统会提示您输入并确认新密码

#### 2. 远程连接

在其他设备上，使用 ssh 命令连接到 archiso 的 IP 地址（例如 10.0.0.5）。如果不知道 IP，可以在 archiso 中运行 ip a 查看

```bash
ssh root@10.0.0.5
```

输入您在 1. 中设置的密码后，您将成功登录并看到类似如下的提示符

```bash
root@archiso ~ #
```

### 1. 更新系统时钟

```bash
# 将系统时间与网络时间进行同步
timedatectl set-ntp true
# 检查时间服务状态
timedatectl status
```

### 2. 磁盘分区与格式化

Surface Pro 5 的原装磁盘为 NVme SSD 设备名一般为 `/dev/nvme0n1`

```bash
# 查看磁盘列表
lsblk
# 使用 cfdisk 进行分区
cfdisk /dev/nvme0n
# 分区结束后检查磁盘情况
fdisk -l
```

以下为我使用的分区方案 (建议根据自身习惯自行调整)
1. **EFI 分区** 大小 512M 类型为 `EFI System`
2. **Swap 分区** 大小 4G 类型为 `Linux swap`
3. **根目录** 剩余所有空间 类型为`Linux filesystem`

建立好分区后需要对分区格式化，这里用 `mkfs.ext4` 命令格式化根分区，用 `mkfs.vfat` 命令格式化 EFI 分区。如下命令中的 `nvme0n1px` 中，x 代表分区的序号。格式化命令要与上一步分区中生成的分区名字对应才可以

```bash
# 格式化 EFI 分区
mkfs.vfat  /dev/nvme0n1px
# 创建并激活 Swap 分区
mkswap /dev/nvme0n1px
swapon /dev/nvme0n1px
# 格式化根分区为 ext4
mkfs.ext4 /dev/nvme0n1px
```

### 3. 挂载磁盘分区

挂载磁盘分区的时候，挂载需要按照顺序，挂载`根目录`后挂载 `EFI 分区`

```bash
# 挂载根目录
mount /dev/nvme0n1px /mnt
# 创建 EFI 目录
mkdir /mnt/boot
# 挂载 EFI 目录
mount /dev/nvme0n1px /mnt/boot
```

### 4. 设置镜像源

使用如下命令设置镜像源

```bash
vim /etc/pacman.d/mirrorlist
```

其中首行是将要使用的镜像源， 对于中国大陆用户添加中科大或者清华的放在最上面即可

```bash
Server = https://mirrors.ustc.edu.cn/archlinux/$repo/os/$arch
Server = https://mirrors.tuna.tsinghua.edu.cn/archlinux/$repo/os/$arch
```

### 5. 安装系统

必须的基础包

```bash
pacstrap /mnt base base-devel linux linux-headers linux-firmware 
```

必须的功能性软件

```bash
pacstrap /mnt vim networkmanager 
# vim 一个文本编辑器
# networkmanager 用于连接无线网络
```

### 6. 生成 fstab 文件

`fstab` 文件定义了系统启动时如何挂载磁盘分区

```bash
genfstab -U /mnt >> /mnt/etc/fstab
```

生成后最好检查一下 `/mnt/etc/fstab`  文件内容是否正确

```bash
cat /mnt/etc/fstab
```

### 7. change root

`chroot` 命令可以让你进入新安装的系统环境，像在真实系统中一样执行命令。

```bash
arch-chroot /mnt
```

### 8. 系统初始化

进了新系统后需要进行一些基础设置 

```bash
# 设置系统时区
ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
hwclook --systohc

# 本地化设置
echo "en_US.UTF-8 UTF-8" >> /etc/locale.gen
echo "zh_CN.UTF-8 UTF-8" >> /etc/locale.gen
locale-gen
echo "LANG=en_US.UTF-8" > /etc/locale.conf

# 设置主机名 (可以将 Surface 替换为自己喜欢的主机名)
echo "Surface" > /etc/hostname

# 配置 hosts 文件
echo "127.0.0.1   localhost" >> /etc/hosts
echo "::1         localhost" >> /etc/hosts
echo "127.0.1.1   Surface.localdomain  Surface" >> /etc/hosts

# 设置 root 用户密码
passwd

# 安装 CPU 微码
pacman -S intel-ucode
```

### 9. 安装引导程序

```bash
pacman -S grub efibootmgr
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=Grub
grub-mkconfig -o /boot/grub/grub.cfg

```

### 10. 安装 linux-surface 内核

使用此内核可以改善触控/手写笔、摄像头、电源管理等支持

```bash
# 导入密钥并信任
curl -s https://pkg.surfacelinux.com/keys.asc | pacman-key --add -
pacman-key --lsign-key 56C464BAAC421453

# 添加软件源
cat >> /etc/pacman.conf <<'EOF'
[linux-surface]
Server = https://pkg.surfacelinux.com/arch/
EOF

# 更新并安装
pacman -Syu
pacman -S linux-surface linux-surface-headers iptsd linux-firmware-marvell

#更新引导程序配置以启用新内核
grub-mkconfig -o /boot/grub/grub.cfg
```

### 11. 完成安装 

注意，重启前要先拔掉优盘，否则你重启后还是进安装程序而不是安装好的系统。重启后，使用 `nmtui` 或 `nmcli` 即可连接网络

```bash
# 退出 chroot 
exit
# 卸载分区 
umount -R /mnt
# 重启
reboot  
```

到此为止，一个无桌面环境的 Arch Linux 在 Surface Pro 5 上就安装完成了
---