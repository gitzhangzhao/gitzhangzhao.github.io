# Petalinux(二)：远程启动

 **申明: "本文严禁任何组织和个人在CSDN上进行转载，其他平台转载需经作者授权。"**

> 对于 Linux 远程启动是很普遍的，在 Petalinux 上配置远程启动很简单，并且 Petalinux 会自动配置 U-Boot 变量并增加内核启动参数

### 1. 目标

- 内核文件通过 tftp 从 host 下载并启动，挂载的根文件系统为网络文件系统（nfs）

- 基本过程：BOOT.BIN 文件包括了硬件 bit 流、FSBL 和 U-Boot，这个文件放在板卡的 tf 卡或者 QSPI Flash 中，用于启动 Linux 内核。Linux 内核和设备树文件 image.ub 放在 host 上的 tftpboot 目录，U-Boot 通过 tftp 下载内核镜像到内存中，并启动内核。内核启动后，挂载 host 上的 nfs 文件系统作为根文件系统

### 2. 组成：FSBL、U-Boot、kernel 和根文件系统

1. **FSBL：** 第二阶段引导文件，由 vivado 提供源码，用于在第一阶段引导代码（固定在芯片内的 rom 里）结束后，初始化一些 ZYNQ 常用的外设，由于是外部代码，可以根据需要自行修改
2. **U-Boot：** 第二阶段引导结束后将跳转执行 U-Boot，fsbl 没有远程拷贝的功能，所以 U-Boot 必须放置在板卡的可启动 ROM 内，主要为 TF 卡和 QSPI FLASH；U-Boot 具有板卡上大部分设备的驱动程序，可以配置网络，下载镜像并拷贝进内存中，最后跳转到内核的启动点
3. **device tree：** 设备树文件描述了硬件设备的信息，需要被 U-Boot 拷贝进内存并告诉内核地址，内核启动后读取设备树文件获取设备信息
4. **Kernel：** 内核镜像，有 uImage、zImage 等不同形式，具有不同的压缩方式与格式。U-Boot 支持各种形式的内核镜像，内核可以在本地部署也可以通过 tftp 或者 nfs 协议从 host 下载内核镜像，内核启动后初始化硬件并挂载根文件系统
5. **根文件系统：** 内核可以挂载本地 eMMC、TF 卡和远程 host 中的文件系统

### 3. BOOT 所需文件的来源

1. **FSBL：** 由 VIVADO SDK 工具提供，使用 make 编译
2. **U-Boot：** 由官方发布源码，Xilinx 添加一些独有驱动，在 Github Xilinx 代码仓库维护，使用 make 编译
3. **Kernel：** Linux 内核源码，使用 make 编译
4. **device tree：** 由内核提供一部分，Xilinx 提供一部分，用户也可以自行添加，使用内核工具中的设备树专用编译器编译
5. **根文件系统：** 不同发行版针对不同体系结构有不同的根文件系统，目录结构固定，直接解压到被挂载的根目录，不需要编译

![请输入图片描述][1]

### 各种文件的组织方式

由于最终都是将各种镜像拷贝进内存，被在每一阶段的最后跳转到下一阶段的执行点，所以理论上 BOOT 的过程可以将各种文件放置在不同的位置，或者相同位置下的不同文件

#### BOOT 文件的打包

1. FSBL、U-Boot、kernel、device tree 一起打包进 BOOT.BIN
2. FSBL、U-Boot 打包进 BOOT.BIN，kernel 和 device tree 打包进 image.ub
3. FSBL、U-Boot 打包进 BOOT.BIN，kernel 和 device tree 独立为 zImage 和 system.dtb
4. 在 2 或 3 的基础上，BOOT.BIN 放在本地，kernel 和 device tree 放在远端，U-Boot 通过 tftp 协议从网络下载 kernel 和 device tree
5. 在 2 或 3 的基础上，BOOT.BIN 放在本地，kernel 和 device tree 放在远端，U-Boot 通过 nfs 协议从网络下载 kernel 和 device tree
6. 在 1、2、3、4、5 的基础上，BOOT.BIN 可以放在 tf 卡的第一分区或者 QSPI Flash 中
7. 在 1、2、3、4、5、6 的基础上，根文件系统可以放在 tf 卡的非第一分区、eMMC 或通过 nfs 从网络挂载

## 4. 使用手动编译生成可远程启动的镜像文件

- 使用手动编译 U-Boot 和 kernel 过程略

  ### 配置 U-Boot 环境变量通过 tftp 启动 linux kernel

  - 在 U-Boot 启动后，通过配置一些 U-Boot 环境变量来远程启动内核，这里假设内核和设备树文件放在 host 的/tftpboot 目录内

```bash
# 设置本机ip地址和host ip地址
zynq> setenv ipaddr 192.168.206.187
zynq> setenv serverip 192.168.206.187
# 将内核镜像zImage和设备树文件system.dtb通过tfpt下载进内存
# 将内核镜像下载到内存地址0x10000000
zynq> tftpboot 10000000 zImage
# 将设备树文件下载到内存地址
zynq> tftpboot 10080000 system.dtb
# 将地址传入bootz命令，启动zImage形式的内核
zynq> bootz 10000000 - 10080000
```

- 由于手动编译设备树和内核文件分离，而挂载文件系统部分相似，后续挂载 nfs 见下一章节介绍

### 5. 使用 petalinux 工具生成可远程启动镜像

在执行 petalinux-config 命令需要修改的部分

1. 1. 修改 U-Boot 配置
      ![请输入图片描述][2]

   2. 可以修改 netboot offset，即从远程下载镜像到内存中的地址偏移，远程 tftp server 的 IP 地址
      ![请输入图片描述][3]

2. 1. 修改镜像打包相关配置
      ![请输入图片描述][4]

   2. 修改根文件系统位置、nfs 文件系统挂载目录、nfs server IP 地址、内核镜像名、以及 host tftpboot 目录
      ![请输入图片描述][5]

3. build 整个系统

   ```bash
   $ petalinux-build
   ```

4. 将编译后的输出文件打包成适合部署的格式

   ```bash
   # 一般 BOOT.BIN 包含 fsbl 文件、bitstream 和 U-Boot 文件
   $ petalinux-package --boot --fsbl --fpga --u-boot --force
   ```

- 其余部分不需要修改，相关根文件系统挂载方式等等配置会生成相关内核参数，内核参数由 U-Boot 在启动时传递给内核

- 这里和之前手动编译生成启动镜像不同，没有下载设备树文件 system.dtb，原因是使用 petalinux 打包的内核镜像 image.ub 中是包含了设备树文件的。因为 image.ub 文件是通过 mkimage 命令制作的，是将内核镜像 zImage 和设备树 system.dtb 打包到一起

### 6. 配置 U-Boot 启动 petalinux 编译的内核

#### 基本步骤

- 基本步骤和手动编译启动内核一致，在远程 host 上需要具有 nfs 服务器，配置好 nfs 目录，并将根文件系统解压进该目录

- 在设置 IP 地址这一步，U-Boot 会先通过 DHCP 从 server 申请 IP 地址，如果 server 上有 DHCP 服务器，U-Boot 就会自动设置 ipaddr 这个环境变量，可以通过 printenv 打印出所有环境变量，serverip 这个环境变量是在 petalinux 配置阶段设置的

- netboot 这个环境变量是 U-Boot 设置的，为 `tfptboot ${netstart} ${kernel_img} && bootm`, 其中 kernel_img 就是 image.ub

- 步骤：

  ```bash
   # 设置本机 ip 地址和 host ip 地址
   zynq> setenv ipaddr 192.168.138.2
   zynq> setenv servimagee通过 tfpt 下载进内存
   # netboot 这个环境变量是 U-Boot 设置的，为 tfptboot ${netstart} ${kernel_img} && bootm, 其中 kernel_img 就是 image.ub
   zynq> run netboot
  ```

#### 配置默认启动命令

- 设置 U-Boot 默认命令，U-Boot 启动后会先配置网络，网络配置完成后就会进入倒计时，倒计时结束执行 bootcmd 中的命令，如果想自动 boot，只需要

  ```bash
  zynq> setenv bootcmd run netboot
  # 保存所有环境变量到 qspi
  zynq> saveenv
  ```

#### 配置内核参数环境变量

- 在设备树文件中有一个 chosen 字段，里面设置了内核参数变量 bootargs，U-Boot 中如果不手动配置这个变量，使用的是设备树文件中的内核参数，但是 U-Boot 中的 bootargs 环境变量具有最高优先级

### 挂载 nfs 文件系统

- 决定根文件系统的方式主要是内核的启动参数，配置从 nfs 挂载，bootargs 应该被设置为

  ```bash
  # petalinux会根据选项自动配置该变量
  zynq> setenv bootargs ‘console=ttyPS0,115200 root=/dev/nfs rw nfsroot=192.168.138.1:/home/zynq/linux/nfs/rootfs,tcp ip=dhcp’
  ```

- 这个环境变量意义是，根文件系统使用 nfs 的方式以 rw 挂载，根文件系统目录在 `192.168.138.1:/home/zynq/linux/nfs/rootfs`，本机 ip 地址配置为 dhcp 方式从 server 获取

- 内核启动后会自动挂载 nfs 文件系统，如果挂载成功，内核就会启动完成

#### 在挂载 nfs 时遇到问题：内核启动报错

  ```bash
  VFS: Unable to mount root fs via NFS, trying floppy.
  VFS: Cannot open root device "nfs" or unknown-block(2,0): error -6
  ```

#### 不能成功挂载的原因

1. 防火墙问题

  ```bash
  $ ufw disable
  ```

2. 由于 nfs 版本不支持

- 目前新安装的 nfs 为 v4 版本，v2 已经废弃，但是很多比较老的内核不支持 v4，可能由于 nfs

#### 解决方法

1. 在内核配置中打开 nfs 文件系统对 nfs v4 的支持：

   `<*> NFS client support for NFS version 4`

   ![请输入图片描述][6]

2. 在 nfs server 端的配置中打开对 nfs v2、nfs v3 和 nfs v4 的支持

  ```bash
  $ echo 'RPCNFSDOPTS="--nfs-version 2,3,4 --debug --syslog"' >> /etc/default/nfs-kernel-server
  $ systmectl restart nfs-server.service
  ```


  [1]: https://pic.imgdb.cn/item/648845061ddac507cc672cbf.png
  [2]: https://pic.imgdb.cn/item/6488459c1ddac507cc6b7b93.png
  [3]: https://pic.imgdb.cn/item/648845c01ddac507cc6c988d.png
  [4]: https://pic.imgdb.cn/item/648845c01ddac507cc6c988d.png
  [5]: https://pic.imgdb.cn/item/648845e31ddac507cc6d7334.png
  [6]: https://pic.imgdb.cn/item/648845e31ddac507cc6d7334.png


