# Ubuntu for TSPI-1M-RK3566 (Linux 6.1)

#### 介绍

为 `TSPI-1M-RK3566` 开发板定制的 Ubuntu 22.04.5 LTS系统，基于Linux 6.1 内核。支持桌面版(xfce)和服务器版两种构建方式。

#### 软件架构

```tree
构建脚本关系图
├── clean-build.sh       # 清理构建环境
├── mk-base-ubuntu.sh    # 生成基础rootfs
├── mk-ubuntu-rootfs.sh  # 安装硬件驱动和软件包
├── mk-image.sh          # 生成可烧录镜像
├── ch-mount.sh          # 挂载/卸载工具
└── post-build.sh        # 镜像后处理脚本
```

#### 安装教程

**环境要求**

* Ubuntu 22.04 LTS 主机环境
* 网络环境确保正常
* 存储空间：至少 `50GB` 可用空间
* 宿主机环境依赖需要安装！具体步骤在下面的 `1. 检查宿主机环境`

**构建步骤**

> 【注意】
> 1. 请确保网络环境良好
> 2. 请确保已经满足所有所主机所需要的环境依赖与软件包等
> 3. 全程都在`ubuntu目录`下操作

1. 检查宿主机环境并安装设置依赖

```bash
sudo apt update \&\& sudo apt full-upgrade \&\& \\
sudo ./host\_check.sh \&\& sudo pip3 install pyelftools \&\& \\
sudo ln -sf /usr/bin/python3 /usr/bin/python \&\& \\
sudo sed -i -e '/\\%sudo/ c \\%sudo ALL=(ALL) NOPASSWD: ALL' /etc/sudoers \&\& \\
sudo usermod -a -G sudo $USER \&\& \\
exec su - $USER
```

2. 选择构建类型（桌面版/服务器版）：

```bash
# 桌面版
GUI=desktop ./mk-base-ubuntu.sh \&\& \\
GUI=desktop ./mk-ubuntu-rootfs.sh \&\& \\
./mk-image.sh

# 服务器版
GUI=console ./mk-base-ubuntu.sh \&\& \\
GUI=console ./mk-ubuntu-rootfs.sh \&\& \\
./mk-image.sh
```

3. 最后会在当前`ubuntu目录`生成 `ubuntu-jammy.img` 镜像文件, 烧录到对应的`rootfs.img`地址即可\~

#### 其他

**清理构建**

```bash
sudo ./clean-build.sh
```

**缺少 deb 包**

> 参考：https://blog.hdochub.com/article/223.html

如果缺少deb包，需要手动下载并放入`SDK目录`下。

需要在 Ubuntu 22 虚拟机中执行：

第一步：生成 kernel 6.1 的标准 deb 包

进入内核源码目录：

```bash
cd /path/to/rk3566_rk3568_linux6.1_release/kernel-6.1
```

执行：

```bash
make CROSS\_COMPILE=aarch64-linux-gnu- ARCH=arm64 \\
LOCALVERSION="" \\
bindeb-pkg -j$(nproc)
```
> 注意加 LOCALVERSION="" 是避免 git hash 附加到版本号（因为 .config 里 CONFIG_LOCALVERSION_AUTO=y）。

生成成功后，会在 SDK 根目录（rk3566_rk3568_linux6.1_release/）出现三个文件：

```bash
linux-headers-6.1.141\_6.1.141-1\_arm64.deb
linux-image-6.1.141\_6.1.141-1\_arm64.deb
linux-libc-dev\_6.1.141-1\_arm64.deb
```

完成后重新编译。

**ubuntu22扩容**

默认的/dev/root大小才5GB，剩余空间仅900MB，完全不够用，因此，需要做扩容分区。

扩容方法就是更改 parameter.txt。

而 rockdev 下的 parameter.txt 是映射的 `tspi-1m-linux-sdk/device/rockchip/rk3566_rk3568/parameter-buildroot-fit.txt`

故修改sdk中的 parameter.txt：

```bash
tspi-1m-linux-sdk/device/rockchip/rk3566_rk3568/parameter-buildroot-fit.txt
```

原本rootfs默认是6G，我们改为12G。

原始内容：
```bash
CMDLINE: mtdparts=:0x00002000@0x00004000(uboot),0x00002000@0x00006000(misc),0x00020000@0x00008000(boot),0x00040000@0x00028000(recovery),0x00010000@0x00068000(backup),0x00c00000@0x00078000(rootfs),0x00040000@0x00c78000(oem),-@0x00cb8000(userdata:grow)
```

关注 CMDLINE 这行，格式为 `size@addr(name)`，其中 size 表示该分区大小，addr 表示该分区起始地址，name 表示分区名。

注意的是，这里以512B为单位的而非1024B，所以 rootfs 的 0x00c00000 是12582912B/1024/1024/512B*1024B = 6GB ，想要修改成12GB就是0x01800000 。

更改为12GB后：

```bash
CMDLINE: mtdparts=:0x00002000@0x00004000(uboot),0x00002000@0x00006000(misc),0x00020000@0x00008000(boot),0x00040000@0x00028000(recovery),0x00010000@0x00068000(backup),0x01800000@0x00078000(rootfs),0x00040000@0x01878000(oem),-@0x018b8000(userdata:grow)
```

完成后先编译debian，然后去kernel-6.1下编译得到deb包，再去ubuntu22编译脚本文件夹下编译出ubuntu22-rootfs.img，最后烧录的时候直接替换你固件位置中的rootfs位置即可。

注意，在烧录替换的时候，要连我们更新的 parameter-fit.txt 也要烧录进去。rootfs就是编译出来的ubuntu22-rootfs.img，oem和userdata还是rockdev下的原始文件。








