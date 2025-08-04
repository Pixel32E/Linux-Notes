# **archlinux安装流程**
声明:本文档由"Pixel32E"于2025.8.4更新如有失效请参考arch官方WIKI社区
## 要安装archlinux发行版你需要做以下准备:

- 首先[下载](https://archlinux.org/download/)官网或者镜像源提供的ISO镜像文件
- 准备一个U盘介质用镜像文件来制作`LiveCD`环境,注意这会删除你U盘的文件请备份!!!
- 可以使用以下软件来制作启动盘:
    - windows平台: 
        - `Ventoy`首选一次制作长期使用多系统爱好者推荐
        - `Rufus`兼容性强
    - Windows/macOS/linux:
        - `balenaEtcher`全平台通用跨平台使用方案
        - `Fedora Media Writer`作为Fedora自家的制作工具因其本身就是linux系统平台兼容性自然是没问题也是跨平台方案支持
        - dd命令linux和macOS可用:注意去掉下文的""号！！！
        ```bash
        sudo dd if="<你的文件>.iso" of=/dev/"你要写入的U盘" bs=4M
        ```

## 这里是目录可以快速定位
1. [连接网络](#连接网络)
2. [更改镜像源](#更改镜像源)
3. [磁盘分区](#磁盘分区)
4. [安装及配置基本系统](#安装及配置基本系统)
5. [安装grub引导](#安装grub引导)
6. [安装基础程序](#安装基础程序)
7. [安装输入法](#安装输入法)

## **连接网络**

### **1. WIFI运行以下命令**
`<SSID>`就是你要连接的wifi名称!;`<device>`是wifi设备的编号例如`wlan0`,`wlan1`.
```bash
iwctl                            # 进入交互式提示符
device list                      # 列出无线设备(如 wlan0)如果你确定自己只有一个无线网卡那么就跳过
station <device> scan            # 扫描网络(station wlan0 scan)
station <device> get-networks    # 列出扫描到的网络
station <device> connect <SSID>  # 连接网络(会提示输入密码)  
exit                             # 退出iwctl
ping archlinux.org               # 测试网络ctrl+c打断ping
```

### **2. 有线直接ping**

```bash
ping archlinux.org # 测试网络ctrl+c打断ping
```

---

## **更改镜像源**

### **1. 用nano或者vim编辑路径 `/etc/pacman.d/mirrorlist`下的文件**

```bash
nano /etc/pacman.d/mirrorlist
```

### **2.nano快捷键:**

- `ctrl+f`=查找"china"在以下镜像源选择你要使用的
- `ctrl+k`=剪切整行
- `ctrl+home键`=快速回到文档顶部
- `ctrl+u`=粘贴 `ctrl+s`=保存 `ctrl+x`=退出(规则是越靠上的优先级越高)

```text
## China
Server = https://mirrors.ustc.edu.cn/archlinux/$repo/os/$arch
Server = https://mirrors.tuna.tsinghua.edu.cn/archlinux/$repo/os/$arch
Server = https://mirror.nju.edu.cn/archlinux/$repo/os/$arch
Server = http://mirrors.aliyun.com/archlinux/$repo/os/$arch
Server = http://mirrors.shanghaitech.edu.cn/archlinux/$repo/os/$arch
```

---

## **磁盘分区**

### **1. 使用磁盘查看命令 `lsblk`或者 `fdisk-l`查看磁盘**

### **2. 磁盘分区使用**

- **`gdisk`或者 `fdisk`**
- **`cfdisk`和 `cgdisk`(有TUI交互对新手友好度高)需要指定磁盘如 `cfdisk /dev/nvme0n1`**

### **3. 分区类型 `DOS/MBR`老旧配置选择不推荐也可以选择 `GPT/UEFI`现代新配置，我们以UEFI为例**

### **4. EXT4及XFS分区方案**

- `EFI(ESP)`引导分区推荐512MiB-1024MiB(1GB)
- 根分区 `/`(系统分区)推荐30~40GB
- 如果有休眠需求 `swap`(交换分区)推荐≥运行内存也可以不分区手动设置页面文件！
- `home`(用户数据)尽量分配大容量看个人使用情况！
- 格式化磁盘及分区推荐使用 `cfdisk`操作(文字交互界面友好)需指定磁盘！

1. **格式化分区(以下为命令操作手搓)**

```bash
mkfs.fat -F32 /dev/nvme0n1p1   # 格式化 EFI 分区为 FAT32(如有Windows引导可以略过直接挂载即可)
mkfs.ext4 /dev/nvme0n1p2       # 格式化根分区为 ext4(或其他如 btrfs, xfs)
# 创建swap分区
mkswap /dev/nvme0n1p4          # 初始化 swap
swapon /dev/nvme0n1p4          # 启用 swap
# 创建home分区
mkfs.ext4 /nvme0n1p3           # 格式化 home 分区(或xfs)
```

2. **挂载分区**

```bash
# 挂载根分区到`/mnt`必须先挂载!
mount /dev/nvme0n1p2 /mnt           
# 创建EFI目录并挂载EFI分区
mkdir -p /mnt/boot/efi
mount /dev/nvme0n1p1 /mnt/boot/efi
# 创建并挂载home分区
mkdir /mnt/home
mount /dev/nvme0n1p3 /mnt/home
```

---

### **5. btrfs分区方案(以下为命令操作为手搓)**

- EFI(ESP)引导分区推荐512MB-1024MB(1GB)
- 尽可能多的空间分配至一个分区格式化为 `btrfs`文件系统
- 进阶用户可以多出一个Boot分区1GB大小或者home分区独立出来 `btrfs`文件系统格式要一致

1. **格式化分区**

```bash
mkfs.fat -F32 /dev/nvme0n1p1         # 格式化 EFI 分区为 FAT32(如有引导可以略过直接挂载即可)
mkfs.btrfs -f /dev/nvme0n1p2         # 格式化分区为btrfs格式 -f 为强制
```

2. **挂载分区及创建子卷**

```bash
mount /dev/nvme0n1p2 /mnt            # 挂载btrfs主分区
```

3. **创建子卷**

```bash
btrfs subvolume create /mnt/@        # 在挂载的/mnt/@ 下创建根分区"@"子卷(子分区)
btrfs subvolume create /mnt/@home    # 在挂载的/mnt/home 下创建用户数据"@home"子卷(子分区)
btrfs subvolume create /mnt/@var     # 在挂载的/mnt/@var 下创建日志文件"@var"子卷(子分区)
btrfs subvolume create /mnt/@swap    # 在挂载的/mnt/@swap 下创建交换分区"@swap"子卷(子分区)
btrfs subvolume create /mnt/@snapshots   # 核心功能快照子卷
```

4. **子卷创建完毕卸载主分区**

```bash
umount /mnt
```

5. **下面开始挂载**

```bash
mount -o compress=zstd,subvol=@ /dev/nvme0n1p2 /mnt # 挂载根子卷并设置"zstd"压缩
mkdir -p /mnt/{home,var,swap,.snapshots,boot}       # 创建挂载目录".snapshots"前面加.是隐藏
mount -o compress=zstd,subvol=@home /dev/nvme0n1p2 /mnt/home
mount -o compress=zstd,subvol=@var /dev/nvme0n1p2 /mnt/var
mount -o compress=zstd,subvol=@snapshots /dev/nvme0n1p2 /mnt/.snapshots # 这里也要带"."
mount -o subvol=@swap /dev/nvme0n1p2 /mnt/swap    # 交换分区不设置压缩
mount /dev/nvme0n1p1 /mnt/boot
```

6. **在@swap子卷创建交换文件**

```bash
cd /mnt/swap    # 先进入目录
dd if=/dev/zero of=swapfile bs=1G count=8 status=progress  # 创建8G交换文件

# 禁用COW（必须操作）
chattr +C swapfile

# 设置权限并初始化
chmod 600 swapfile
mkswap swapfile
swapon swapfile
```

---

## **安装及配置基本系统**

### 1. 使用 `pacstrap` 脚本安装最小化的基础系统到 `/mnt`

- `base`是核心系统包
- `base-devel`是编译和构建工具(如 `make`, `gcc`)
- `linux`是默认内核(可选内核 `linux-lts`, `linux-zen`)
- `linux-firmware`是硬件固件

```bash
pacstrap -K /mnt base base-devel linux linux-firmware
```

### 2. 配置系统

**生成 fstab 文件(定义挂载点)**

```bash
genfstab -U /mnt >> /mnt/etc/fstab
```

检查生成的 `/mnt/etc/fstab` 是否正确

```bash
cat /mnt/etc/fstab
```

### 3. chroot 到新系统

```bash
arch-chroot /mnt
```

### 4. 本地化设置

1. 使用nano编辑 `/etc/locale.gen`

```bash
nano /etc/locale.gen    # 取消"#"注释
```
取消"#"注释以设置需要的语言环境如 `en_US.UTF-8 UTF-8`, `zh_CN.UTF-8 UTF-8`,`UTF-8`

2. 应用语言环境设置

```bash
locale-gen # 应用语言文件会输出注释掉的语言选项
```

3. 设置系统语言环境变量，用echo快速添加文本到 `/etc/locale.conf`

```bash
echo "LANG=en_US.UTF-8" > /etc/locale.conf
```

### 5. 设置主机名

**编辑 `/etc/hostname`，写入你想要的名称例如:"我的电脑"**

```bash
echo "这里写名字不要用非英文的字符" > /etc/hostname
```

**编辑 `/etc/hosts`，添加 # 替换以下 `myarch` 为你的主机名**

```text
127.0.0.1   localhost
::1         localhost
127.0.1.1   这里填写你上一步设置的名字.localdomain 这里同样是你设置的名字
```

### 6. 设置账户 `<username>`是你的用户名

```bash
passwd    # 设置root的密码
useradd -m -G wheel -s /bin/bash <username>  #添加账户
userdel -f <username>    # 删除账户 -r 文件夹也一起删除！！
passwd <username>        # 设置用户密码
```

### 7. 账户提权sudo用nano编辑 `/etc/sudoers`

```bash
nano /etc/sudoers
```
把`# %wheel ALL=(ALL:ALL) ALL`前面的#号和空格删除让它变成这样: `%wheel ALL=(ALL:ALL) ALL`保存退出

### 8. 安装微码

```bash
pacman -S amd-ucode         # AMD  处理器
pacman -S intel-ucode       # Intel处理器
```

---

## **安装grub引导**

```bash
pacman -S grub efibootmgr os-prober
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=GRUB
```

输出 `No error repoted`就是成功了

### 1. 生成配置文件

这一步是为了生成文件给下面的步骤进行编辑配置，刚安装默认是没有配置文件的所以要先生成一个！
```bash
grub-mkconfig -o /boot/grub/grub.cfg
```

### 2. 编辑grub配置(双系统)单系统直接跳过该步骤和下面的步骤

```bash
nano /etc/default/grub
```
找到最后一行 `#GRUB_DISABLE_OS_PROBER=false` 注释掉#号保存退出即可

### 3. 再次重新生成配置文件

```bash
grub-mkconfig -o /boot/grub/grub.cfg
```
重新生成配置文件后你的Windows的引导就被grub找到并添加到启动菜单了

## **安装基础程序**

### 1. 安装网络管理器

有线网络
```bash
pacman -S dhcpcd #安装程序
systemctl enable dhcpcd.service #启动服务
```

WIFI网络:

1. 使用`iwd`适用于窗口管理器环境注意如果同时安装`networkmanager`会冲突导致`iwd`失效

```bash
pacman -S iwd #无线网络程序
systemctl enable iwd #启动服务
systemctl start iwd #启动
```
2. 使用`NetworkManager`是桌面环境常用的网络管理器推荐使用这个
```bash
pacman -S networkmanager  #无线网络
systemctl enable NetworkManager   # 设置启动注意字母大小写！
```
3. 冲突解决方法停止并禁用: `NetworkManager`和`wpa_supplicant`

```bash
sudo systemctl stop NetworkManager
sudo systemctl disable NetworkManager
sudo systemctl stop wpa_supplicant
sudo systemctl disable wpa_supplicant
```
然后在按照上面的步骤重新使用iwd

### 2. 安装桌面(desktop) 不要装两个!!!

- **[GNOME](https://www.gnome.org/)桌面**

```bash
pacman -S gnome gdm             # 桌面
systemctl enable gdm               # 启动图形管理器
```

- **[KDEplasma](https://kde.org/zh-cn/plasma-desktop/)桌面**

```bash
pacman -S plasma-meta sddm kde-applications xorg                    # 桌面和依赖
systemctl enable sddm              # 启动图形管理器
```

- **KDEplasma桌面精简版包含核心组件**

```bash
sudo pacman -S plasma-desktop sddm dolphin konsole plasma-nm plasma-pa kscreen
sudo pacman -S plasma-system-meta   # 这个包含以上plasma-nm plasma-pa kscreen plasma-desktop kde核心应用推荐安装（极致精简可跳过）
sudo pacman -S systemctl enable sddm      # 启动图形管理器
```

- **plasma-desktop**: 最基础的 KDE 桌面环境（不含完整应用套件）
- **sddm**: 轻量级登录管理器
- **dolphin**: 必备文件管理器
- **konsole**: KDE 终端
- **plasma-nm**: 网络管理组件
- **plasma-pa**: 音频控制组件
- **kscreen**:显示器配置

### 3. 安装工具（按需选择）

```bash
sudo pacman -S ark kate spectacle gwenview timeshift nano vim 
```

- **ark**: 压缩文件管理(Kde)
- **kate**: 高级文本编辑器(kde)
- **spectacle**: 截图工具(kde)
- **gwenview**: 图片查看器(kde)
- **timeshift** 分区快照工具(备份通用)
- **nano,vim** 终端文本编辑器(必备且通用)

### 4. 安装字体

```bash
sudo pacman -S noto-fonts noto-fonts-cjk noto-emoji ttf-dejavu
sudo pacman -S wqy-zenhei wqy-microhei
```

### 5.退出chroot并重启

```bach
exit
reboot
```

## **安装输入法**

### 选择要安装的输入法，从以下两种输入法选择一个

1. **ibus输入法**

```bash
sudo pacman -S ibus ibus-libpinyin
```

2. **fcitx5输入法**

```bash
sudo pacman -S fcitx5-chinese-addons fcitx5-configtool fcitx5-gtk fcitx5-qt
sudo pacman -S fcitx5-pinyin-zhwiki
```

### [如果使用Wayland](https://fcitx-im.org/wiki/Using_Fcitx_5_on_Wayland#GNOME)需要给fcitx5输入法设置环境变量在终端中输入就行

```bash
XMODIFIERS=@im=fcitx
QT_IM_MODULE=fcitx
GTK_IM_MODULE=fcitx
```

### GNOME桌面[点击安装拓展启用输入法图形](https://extensions.gnome.org/extension/261/kimpanel/)

### GNOME桌面需安装 `gnome-tweaks`拓展，设置输入法开机自启动

```bash
sudo pacman -S gnome-tweaks
```

### 重启 `reboot`

```bash
reboot
```
### 恭喜你坚持到这里并完成以上所有步骤成功解锁成就: 安装atchlinux
### 如果你对linux感兴趣那么arch是你快速入门很好的选择！快去体验吧！
