# con-file
Archlinux下的一些配置文件
## 基础软件
### 终端模拟器：Alacritty
### 输入法：fctix5-nord
### 显卡管理：optimus-manager
### 文件管理器：dolphin+ranger
### 终端：zsh+powerlevel10k
### 核心组件平替：ls,find,cd...[参考](https://wiki.archlinux.org/title/Core\_utilities#Essentials)
### 回滚：timeshift
### 日志管理：syslog-ng

## 办公
### office：okular+libreoffice
### V2rayA
### tmux 终端后台运行

## 日常娱乐
### QQ：icalingua++
### 音乐：listen1
### 浏览器：firefox
### 截图工具：flameshot


# Arch linux 下我所遇到的问题
首先说，保存快照是一个好习惯。基本的问题都可以通过timeshift回滚解决。
## Q: -> Failed to install the following packages. Manual intervention is required:    
```zsh
yay -Scc Cache

directory: /var/cache/pacman/pkg/ :: Do you want to remove ALL files from cache? [y/N] n
Database directory: /var/lib/pacman/ :: Do you want to remove unused repositories? [Y/n] n
Build directory: /home/joe/.cache/yay ==> Do you want to remove ALL AUR packages from cache? [Y/n] y 
removing AUR packages from cache...
```
## Q: ->  The engine is incompatible with this module. Expected version "8 | 10 || 12 || 14 || 16 || 17". Got "20.0", error Found incompatible module. 
```zsh
yay -S nvm 
source /usr/share/nvm/init-nvm.sh
echo 'source /usr/share/nvm/init-nvm.sh' >> ~/.bashrc
echo 'source /usr/share/nvm/init-nvm.sh' >> ~/.zshrc
nvm install [--version]
node --version 
```
## Q: how to install cuda + pytorch (READ [NVIDA-GUIDE](https://docs.nvidia.com/cuda/cuda-installation-guide-linux/))    
安装及配置：
```zsh
sudo pacman -S python-pytorch-cuda # 注意区别python-pytorch
sudo pacman -S cuda
```    
可能用到的其他指令
```zsh
uname -m && cat /etc/*release # 查看系统
gcc --version # 查看gcc版本检测是否安装
g++ --version
which nvcc # 查找cuda路径
cuba-gdb # 检测cuba是否安装成功
nvidia-smi # 查看当前显卡驱动及其支持的cuda版本
```

## Q: Loading Linux linux... error: invalid cluster 0, error: you need to load the kernel first.     
这里显然是linux  kernel 出了问题，首先准备系统盘，进入外接系统，执行
```zsh
# 无线连接，插网线可跳过该步
iwctl
device list
station wlan0 scan
station wlan0 get-networks
station wlan0 connect wifi-name
exit # 连接成功后推出
ping www.bilibili.com # 测试网络连通性
```

```zsh
# 首先，将损坏的linux系统重新挂载
mount -t btrfs -o subvol=/@,compress=zstd /dev/nvmexnxpx /mnt # 这里根据实际情况选择nvme和sda，分区为linux主要存储分区
mkdir /mnt/home # 报错已存在为正常现象
mount -t btrfs -o subvol=/@home,compress=zstd /dev/nvmexnxpx /mnt/home # 同上一个分区
mkdir -p /mnt/boot
mount /dev/nvmexn1p1 /mnt/boot # 仍然nvme or sda 取决实际情况，分区为efi系统分区，区别于前两个。
df -h #查看挂载情况
```

```zsh
# 挂载后切换至损坏系统
arch-chroot /mnt
pacman -Syy 
yay
pacman  -S linux linux-wirmfare  # 重装linux kernel
exit # 推出损坏系统
umount -R /mnt # 卸载新分区
reboot #重启并 拔下U盘 
```
在理想情况下，进行以上操作后就不会有其他问题了。但这个过程中也可能伴随其他的问题。
### Q: cannot create regular file '/boot/vmlinuz-linux': Read-only file system. Error: command failed to execute correctly
简单来说遇到*Read-only*问题只要把文件所在分区重新挂载一下，允许写入即可。
```zsh
mount -o remount -rw /data
```
### Q: Error: failed to setup chroot /mnt    
  经常伴随上一个问题中``arch-chroot /mnt``时出现，``lsblk``检查当前分区情况，``df -h``检查挂载情况。如有问题重新挂载。确认挂载没有问题执行如下：
```zsh
pacstrap /mnt  base linux  linux-firmware
arch-chroot /mnt
```

## Q:  Error: "no filesystem could mount root"  OR "failed to  mount /boot"
这显然是grub出现了问题，同样，你需要准备一个系统盘
```zsh
lsblk
mkfs.vfat -F32  /dev/nvmexnxpx # 你的efi分区。注意，这里格式化efi分区，若为双系统，推荐略过。
mount -t btrfs -o subvol=/@,compress=zstd /dev/nvmexnxpx /mnt # 这里根据实际情况选择nvme和sda，分区为linux主要存储分区
mkdir /mnt/home # 报错已存在为正常现象
mount -t btrfs -o subvol=/@home,compress=zstd /dev/nvmexnxpx /mnt/home # 同上一个分区
df -h # 检测，若为双系统尤其注意此时efi系统分区剩余空间是否充足(>100M)
arch-chroot /mnt # 进入损坏系统
```
```zsh
# 重新生成引导分区
pacman -S grub # 可选，一般不需要重装
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB
grub-mkconfig -o /boot/grub/grub.cfg
cat /boot/grub/grub.cfg # 查看是否正常生成
```
若grub.cfg没有生成，输入``ls /boot``查看是否有以下文件    
```
initramfs-linux.img
intel-ucode.img or amd-ucode.img
vmlinuz-linux
```

如果没有,则执行
```zsh
pacman -S linux
grub-mkconfig -o /boot/grub/grub.cfg
ls /boot # 可以看到三个文件
```

```zsh
exit # 推出损坏系统，回到系统盘
rm -rf /mnt/etc/fstab # 删除旧的挂载文件
genfstab -U /mnt >> /mnt/etc/fstab # 重新生成
```

```zsh
# 检查挂载与硬盘UUID是否一致，这里如果fstab中使用名称建议更换为UUID
ls -l /dev/disk/by-uuid
cat /mnt/etc/fstab
exit
reboot # 并拔出U盘
```
