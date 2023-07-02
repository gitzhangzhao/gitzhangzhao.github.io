# Petalinux(三)：完全使用手册


**_申明: 本文严禁任何组织或个人在 CSDN 上进行转载,其他平台转载需经作者授权_**

> 之前的两篇文章介绍了 Petalinux 的安装和远程启动，本手册是系列的第三篇文章。本篇以 Embedded EVR 为例，详细介绍从 FPGA 代码的综合、布局布线、生成 bit 流，到使用 Petalinux 创建工程、配置、编译和部署的过程

> 前提条件：一台可以联网的服务器，并且已经安装好 Vivado 软件

## 前言

博士期间，我主要利用 timing4(192.168.206.186) 服务器完成嵌入式 EVR 的研究，其中涉及到 FPGA 开发和嵌入式 Linux 系统。嵌入式系统的开发往往具有步骤复杂、过程繁琐的特点，Petalinux 这类自动化的软件工具大大简化了开发步骤

在 Docker 环境中使用 Petalinux 的原因，已经在系列第一篇文章中介绍了。本手册按照如下的步骤来介绍 Petalinux 的使用：

- 在 timing4 上生成嵌入式 EVR 的 bit 流文件
- 导出嵌入式 EVR 的硬件描述文件到 Petalinux 容器中
- 制作 Petalinux 镜像，以 2017.04 为例
- 使用 Petalinux 镜像创建容器
- 修改或添加所需的设备树
- 根据硬件描述文件配置 Petalinux
- 配置 Linux 内核、U-Boot 和根文件系统
- 编译整个 Petalinux 工程
- 打包并整理内核、U-Boot 和根文件系统等镜像
- 使用镜像文件制作可启动的 TF 卡
- 容器使用中可能遇到的问题
- 在遇到不可解决的问题时，恢复 Docker 环境

## 内容

### 在 timing4 上生成嵌入式 EVR 的 bit 流文件

#### 1. 登录到 timing4 服务器的图形界面

由于我们需要使用 Vivado 工具，所以需要登录图形界面。登录 timing4 服务器的图形化界面有两种方式：

- 使用 xrdp 协议
  timing4 服务器安装了 xrdp 服务器，可以直接使用 windows 自带的远程桌面，登录 192.168.206.186.用户名使用自己的用户名，随后会提示输入密码

  ![请输入图片描述][0]

- 使用 rustdesk
  timing4 服务器上安装了 rustdesk Server，远程登录需要在 timing4 上安装运行 rustdesk Client，并且在自己的电脑上也安装好 rustdesk Client。具体的操作步骤请参考[rustdesk 使用手册](https://rustdesk.com/docs/zh-cn/)

上述两种方式各有优劣，方式 1 不需要安装额外客户端软件，方式 2 具有文件传输等更丰富便捷的功能

#### 2. 启动 Vivado

timing4 上的 Vivado 安装在目录`/opt/Xilinx/Vivado/2017.4`下，在这个目录下存在 setting64.sh 文件，其中配置了 Vivado 所需的一些环境变量以及 PATH 变量。建议在启动 Vivado 之前先执行`source /opt/Xilinx/Vivado/2017.4/setting64.sh`，也可以将这个命令添加进用户自己的.bashrc 或者.zshrc 中。.zshrc 中的环境变量可以参考`/home/zhangzh/.zshrc`

在终端中执行 vivado 命令，即可打开 Vivado 的图形化界面，此后的操作和 Windows 下一致：

![请输入图片描述][1]

#### 3. 在 Vivado 中综合并布线嵌入式 EVR

嵌入式 EVR 的代码放在目录`/home/zhangzh/Lab/vivado`中，其中 test9 为 EVR 功能模块的代码，打包为自定义 IP，放在`/home/zhangzh/Lab/vivado/test9/test9.ip_user_files`目录下。test9_wizard 为自定义 IP 和 ZYNQ IP 的连线工程，其主要内容为 Block Design 和约束文件

![请输入图片描述][2]

如果没有修改代码的需要，则只需打开 test9_wizard 工程。在下图 1 的位置有生成 bit 流的按钮，将按顺序自动的运行综合，布局布线，最后产生 bit 流。一杯咖啡后，右上角显示`wirte_bitstream Complete √`即表示运行成功。布线后生成的 bit 流文件为`/home/zhangzh/Lab/vivado/test9_wizard/test9_wizard.runs/impl_1/design_1_wrapper.bit`。这个文件可以通过 JTAG 的方式直接烧写进 FPGA 中，但无法被 U-Boot 或 xdevcfg 烧写。还需要使用 bootgen 工具将其转换为 BIN 文件格式

![请输入图片描述][3]

#### 4. 将硬件描述文件导出

为什么上一步已经得到了嵌入式 EVR 的 bit 流，还需要将硬件描述文件导出？这是因为硬件描述文件(.hdf)描述了完整的硬件信息，包括 FPGA 中使用的资源，这些信息正是 Petalinux 所需要的。需要注意的是，在 Vivado 的高版本中，hdf 文件已经被 xsa 文件替代

![请输入图片描述][4]

此时，FPGA 部分的工作就可以结束了，Petalinux 随后会根据硬件描述文件中提供的信息去配置工程，或者从 Vivado 工程中将 bit 流文件拷贝出来。在 Petalinux 最终生成的镜像文件中，同样也包含 bit 流文件

### 制作 Petalinux 镜像

本节以 2017.04 版本为例，介绍如何在 timing4 上制作 Petalinux 镜像

1. 创建 Dockerfile 文件

```sh
$ touch Dockerfile
# 将以下内容写入 Dockerfile
$ cat Dockerfile
FROM ubuntu:16.04
RUN dpkg --add-architecture i386
RUN apt update \
        && apt install -y sudo libssl-dev flex bison chrpath socat autoconf libtool texinfo gcc-multilib libsdl1.2-dev libglib2.0-dev screen pax net-tools wget diffstat xterm gawk xvfb git make libncurses5-dev tftpd zlib1g libssl-dev gnupg tar unzip build-essential libtool-bin dialog cpio lsb-release zlib1g:i386 zlib1g-dev:i386 locales openjdk-8-jdk"

# build 镜像
$ docker build -t <image_name> .
```

使用 docker images 命令来管理镜像。latest 为默认的镜像 tag

![请输入图片描述][5]

### 使用 Petalinux 镜像创建容器

获得镜像后，创建一个容器

```sh
# 使用镜像创建容器
$ docker run -it --name <name> -v <dir:dir> --user root <image_name> /bin/bash

# 使用 docker exec 命令启动 docker
$ docker start name
$ docker exec --user root -it <name> /bin/bash
```

进入容器后，在容器中继续执行安装

```sh
# 添加普通用户
$ adduser <user_name>

# 修改 /etc/sudoers 文件，添加 sudo 权限

# 用普通用户身份启动 docker
$ docker exec --user <user_name> -it <name> /bin/bash

# 配置安装 locales， 使用 en_US.UTF-8
$ sudo dpkg-reconfigure locales

# 设置 LANG 环境变量为 en_US.UTF-8
$ export LANG="en_US.UTF-8"

# 从共享目录中将 Petalinux 安装镜像 copy 到容器中

# 执行安装，lisence 输入 yes
$ ./petalinux-v2017.4-final-installer.run dir

# 安装完成后，source Petalinux 的安装目录中的 setting.sh

# petalinux 开头的命令可以使用
$ petalinux-xxx
```

使用 docker ps -a 查看所有的容器
![请输入图片描述][6]

其中 petalinux_image 创建了 2 个容器，这两个容器是相关不关联的

注意：mzz2017/v2raya 镜像和 v2ray 容器是 timing4 服务器上的系统代理，用于访问外网。在浏览器中输入`localhost:2017`，管理代理的节点

### 修改或添加所需的设备树

假设 petalinux_20220621 容器已被成功创建

timing4 每次重启后，都需要运行`docker start petalinux_20220621`来启动容器。然后使用`docker exec --user zhangzh -it petalinux_20220621 /bin/bash` 命令来进入容器，`--user`后应为自己在容器中新建的用户

需要注意的是：Petalinux 不支持除了 bash 以外的其他 shell

为了方便每次 source Petalinux 的安装目录中的 setting.sh，可以在自己的目录下的`.bashrc`文件中添加如下内容：

```sh
$ tail -3 .bashrc
export LANG="en_US.UTF-8"
cd $HOME
source /xxx/settings.sh
```

[0]: https://pic.imgdb.cn/item/64a0ebcb1ddac507cc4df043.jpg
[1]: https://pic.imgdb.cn/item/64a0eab41ddac507cc4c5225.jpg
[2]: https://pic.imgdb.cn/item/64a0eb2a1ddac507cc4cff4f.jpg
[3]: https://pic.imgdb.cn/item/64a0e9331ddac507cc49ee63.png
[4]: https://pic.imgdb.cn/item/64a0eb8c1ddac507cc4d9473.jpg
[5]: https://pic.imgdb.cn/item/64a0f9681ddac507cc63e20a.jpg
[6]: https://pic.imgdb.cn/item/64a0fa761ddac507cc657d99.jpg

