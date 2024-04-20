## 4G Modem Stick (MSM8916) Linux 编译与安装 

- 适用于芯片为 msm9816 的 4G Modem Stick 也就是骁龙410

### 编译Linux源码

#### 安装软件包

```shell
sudo apt install gcc make git
sudo apt install binfmt-support gcc-aarch64-linux-gnu build-essential fakeroot mkbootimg bison flex libncurses-dev
sudo apt install debhelper-compat bc libelf-dev:native libssl-dev:native rsync
```
#### 克隆源代码

```shell
git clone -b ${{ KERNEL_BRANCH }} https://github.com/msm8916-mainline/linux --depth=1
# KERNEL_BRANCH: 对应的内核分支，注意分支有没有包含设备树
```

#### 修改环境变量

```shell
export ARCH=arm64
export CROSS_COMPILE=aarch64-linux-gnu-
```

#### 配置内核

```shell 
# 使用默认配置
make msm8916_defconfig
# 在图形化界面选择自定义配置
make menuconfig
```

#### 编译

```shell
make LOCALVERSION=-msm8916 -j12 deb-pkg
# LOCALVERSION: 编译完内核后面会带上这个
# -j: 线程数 我的CPU是12线程的我选12
# deb-pkg: 生成deb格式的安装包适合debian和ubuntu安装
```



### 安装内核

**编译生成的.deb在linux源代码的上级目录**

#### 直接安装

将所有的.deb文件拷贝到设备中直接安装即可

```shell
# 所有的.deb复制到设备上之后在目录中运行
dpkg -i *.deb
```

#### 在rootfs中安装(x86环境)

- **安装模拟环境**

  ```shell
  sudo apt-get install qemu-user-static binfmt-support
  ```

- **确保模拟环境正常运行**

  ```shell
  # 检查模拟环境是否正常运行
  ls /proc/sys/fs/binfmt_misc
  
  # WSL默认设置情况下在安装binfmt-support后不会启动服务
  # 如果没有成功启动，需要手动启动服务
  sudo service binfmt-support start
  ```

- **挂载rootfs镜像**

  ```shell
  # 新建rootfs目录
  mkdir rootfs
  
  # rootfs转换
  simg2img rootfs.img root.img
  # 如果没有simg2img，可以使用 apt search simg2img 找到对应的软件包安装即可
  
  # 挂载镜像
  sudo mount root.img rootfs/
  
  # 复制qemu
  sudo cp /usr/bin/qemu-aarch64-static rootfs/usr/bin/
  ```

- **挂载本地设备**

  ```shell
  sudo mount -t proc	/proc		rootfs/proc
  sudo mount -t sysfs	/sys		rootfs/sys
  sudo mount -o bind	/dev 		rootfs/dev
  sudo mount -o bind	/dev/pts	rootfs/dev/pts
  ```

- **安装内核**

  ```shell
  # 将.deb文件全部复制搭配rootfs里的tmp目录中
  # 切换路径
  chroot rootfs
  cd /tmp
  
  # 安装内核
  # 在安装内核之前可以先卸载旧的内核文件 apt remove ${OLD_KERNEL}
  dpkg -i *.deb
  
  # 大部分rootfs进行都进行过大小调整，若安装过程中出现空间不足，重新调整一下大小即可
  ```

- **取消挂载**

  ```shell
  # 取消挂载本地设备
  sudo umount rootfs/dev/pts
  sudo umount rootfs/dev
  sudo umount rootfs/proc
  sudo umount rootfs/sys
  # 取消挂载rootfs分区
  sudo umount rootfs/
  
  simg2img root.img rootfs.img
  ```

#### 在rootfs中安装(Arm 环境)

你既然都在Arm环境编译内核了，应该就不许要教程了吧？不需要了吧？

省去x86环境下**安装模拟环境**和**复制qemu**步骤其他几乎一样



### boot.img 制作

#### initrd 生成

```shell
# .deb安装完成后制作initrd(如果你在/boot目录中生成了版本号对应的initrd可以跳过此步骤)
mkinitramfs -o initrd ${KERNEL_VERSION}
# KERNEL_VERSION可以在/lib/modules中看到
# 如果没有mkinitramfs可以apt search mkinitramfs找到对应的软件包安装
```

#### 复制准备制作boot的文件

```shell
# 回到linux源代码的上层目录，就是生成.deb文件所在的目录
# 复制文件
cp linux/arch/arm64/boot/Image.gz ./
cp linux/arch/arm64/boot/dts/qcom/${DEVICE_DTB} ./
# DEVICE_DTB是设备的设备树名称
```

#### boot.img 制作

```shell
# 设备树
cat Image.gz ${DEVICE_DTB} > kernel-dtb

# 将之前生成的initrd文件拷贝到当前目录
# boot制作
mkbootimg \
    --base 0x80000000 \
    --kernel_offset 0x00080000 \
    --ramdisk_offset 0x02000000 \
    --tags_offset 0x01e00000 \
    --pagesize 2048 \
    --second_offset 0x00f00000 \
    --ramdisk initrd \
    --cmdline "earlycon root=PARTUUID=a7ab80e8-e9d1-e8cd-f157-93f69b1d141e console=ttyMSM0,115200 no_framebuffer=true rw"\
    --kernel kernel-dtb -o boot.img
```

#### FASTBOOT 刷机

刷入boot和rootfs即可



## Debian rootfs 制作

**从零开始为Arm64设备制作一个rootfs**



### 构建根目录(x86)

在x86架构的设备上制作arm64架构的rootfs

#### 环境安装

```shell
sudo apt-get install qemu-user-static binfmt-support debootstrap
```

#### 下载根文件系统

```shell
sudo debootstrap --arch=arm64 --foreign --include=whiptail,ca-certificates,tzdata bookworm debian http://ftp.cn.debian.org/debian/
```

#### 设置qemu

```shell
sudo cp /usr/bin/qemu-aarch64-static debian/usr/bin/
```

#### 确保binfmt-support服务已经启动

```shell
ls /proc/sys/fs/binfmt_misc

# 我目前用的WSL默认设置情况下在安装binfmt-support后不会启动服务
# 如果没有成功启动，需要手动启动服务
sudo service binfmt-support start
```

#### 安装

````shell
# 切换目录
sudo chroot debian/

# 安装
debootstrap/debootstrap --second-stage
````

#### 挂载本地设备

```shell
sudo mount -t proc	/proc		debian/proc
sudo mount -t sysfs	/sys		debian/sys
sudo mount -o bind	/dev 		debian/dev
sudo mount -o bind	/dev/pts	debian/dev/pts
```

#### 下载软件包

```shell
apt update
apt upgrade
apt install systemd systemctl systemd-timesyncd -y
apt install udev dbus sudo curl locales network-manager net-tools wireless-regdb wireless-tools openssh-server openssh-client libusbgx2 -y
#还可以安装其他的工具和进行systemd服务配置
```

#### 取消挂载

```shell
umount /dev/pts/ /dev/ /proc/ /sys
```



### 制作rootfs.img

#### 创建img文件

```shell
dd if=/dev/zero of=root.img bs=1M count=2000
# dd:	是linux命令，用于读取、转换并输出数据。
# if:	表示输入文件。
# of:	表示输出文件。
# bs:	表示块大小。
# count:	表示被复制的块数。

mkfs.ext4 -F -L linuxroot root.img
# mkfs.ext4 ：为linux命令，用于对磁盘设备进行Ext4格式化的操作。
# -F ：表示强制格式化。
# -L ：设置卷标。这里则是linuxroot
# rootfs.img : 则是设备
```

#### 创建文件夹并挂载

```shell
mkdir rootfs
sudo mount root.img rootfs/
```

#### 复制文件

```shell
sudo cp -rfp debian/* rootfs/
```

#### 卸载并检查

```shell
sudo umount rootfs/

sudo e2fsck -p -f root.img
检查ext4文件系统的正确性。上述命令中：
-f：表示即使文件系统没有错误迹象，仍强制地检查正确性。
-p：表示不询问使用者意见，便自动修复文件系统。
```

#### 调整文件大小

```shell
sudo resize2fs -M root.img
```



> [msm8916-linux - https://github.com/msm8916-mainline/linux](https://github.com/msm8916-mainline/linux)
>
> [制作aarch64/arm64 Debian rootfs系统](read://https_blog.csdn.net/?url=https%3A%2F%2Fblog.csdn.net%2FZempy0513%2Farticle%2Fdetails%2F126843501)
>
> [bootfs制作](read://https_blog.csdn.net/?url=https%3A%2F%2Fblog.csdn.net%2Firiczhao%2Farticle%2Fdetails%2F127078414%23%3A~%3Atext%3D%25E5%259C%25A8%25E7%25BB%2588%25E7%25AB%25AF%25E8%25BE%2593%25E5%2585%25A5%25EF%25BC%259A%2520sudo%2520mkfs.ext4%2C-F%2520-L%2520linuxroot%2520rk3568.img)



