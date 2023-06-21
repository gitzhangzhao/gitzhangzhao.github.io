# Gentoo 安装小记


**_申明: 本文严禁任何组织或个人在 CSDN 上进行转载，其他平台转载需经作者授权_**

### 前言

> 安装 Gentoo 并不复杂，很多人混淆了复杂和耗时。在安装的过程中，大部分的时间都在做别的事情。同时，Gentoo 的安装步骤是清晰的，Handbook 和各路神仙的总结实际上已经很全面了。因此，没有必要再做重复的劳动，一些个性化的关键点记录就足够了。

> 为了使系统保持 Suckless，尽量避免用不到的功能，我还是继续沿用裸 wm 的方式。简而言之：gentoo + openrc + i3wm + polybar + nvim。我的目标是尽量在一天内完成系统的整体安装，再用一周时间进行小修小补。而在流程化的步骤下，一天内的实际安装时间在 1 小时左右，而其余时间都在做其他事情。

> 此外，对于小新 Pro 这种散热垃圾的 Laptop，一个带风扇的散热架是必要的。否则，`emerge -e @world` 是真的会卡死（哭）。

### 安装步骤

Gentoo 的安装大体上是规范的，但是针对不同用户的需求和理念，也有不一样的方式。或多或少的，存在一些坑

我在安装过程主要参考的几个链接：

1. 官方 Handbook，这是最权威的手册，当问题不确定时，以 Handbook 为准

   [Handbook](https://wiki.gentoo.org/wiki/Handbook:AMD64/zh-cn)

2. 一篇较为详细的安装笔记，属于经验丰富的老玩家心得了，很有参考价值

   [Gentoo 安装流程分享(step by step)，第一篇之基本系统的安装](https://zhuanlan.zhihu.com/p/122222365)

3. OriPoin's blog，详细介绍了 Gentoo 的优化方式，但是没必要采用 O3，会带来很多未知问题

   [Emerge your world the lean way](https://blog.oripoin.me/2022/04/emerge-your-world-the-lean-way/)

   [Optimize Your system the stupid way](https://blog.oripoin.me/2022/04/optimize-your-system-the-stupid-way/)

4. bitbili's blog，非常非常详细的介绍了 Gentoo 的安装和使用

   [Gentoo Linux 安装及使用指南](https://bitbili.net/gentoo-linux-installation-and-usage-tutorial.html)

5. Yangmame 的博客（比较早期）

   [Gentoo 安装教程](https://blog.yangmame.org/Gentoo%E5%AE%89%E8%A3%85%E6%95%99%E7%A8%8B.html)

6. ayamir 的知乎记录（参考了 2 和 5）

   [2020-Gentoo 双系统安装指北](https://zhuanlan.zhihu.com/p/166652475)

7. GTrush 的博客

   [新手 Gentoo 折腾记录 1](gtrush.com)

8. Jioushan 的博客

   [不完整的 Gentoo 安装](blog.jsmsr.com)

9. Google，Stack Overflow，gentoo wiki，arch wiki 等

### make.conf

make.conf 可以说是 Gentoo 的核心了，针对 PC 的配置、优化以及对系统的预期基本上都是在这个文件中定义的，首先列出我的：

```make.conf
 # These settings were set by the catalyst build script that automatically
 # built this stage.
 # Please consult /usr/share/portage/config/make.conf.example for a more
 # detailed example.
 COMMON_FLAGS="-march=native -O2 -pipe"
 CFLAGS="${COMMON_FLAGS}"
 CXXFLAGS="${COMMON_FLAGS}"
 FCFLAGS="${COMMON_FLAGS}"
 FFLAGS="${COMMON_FLAGS}"

 USE="X elogind mount cjk i3wm mpd network pulseaudio ipc opengl dbus -gnome -kde"

 MAKEOPTS="-j6"
 LC_MESSAGES=C
 EMERGE_DEFAULT_OPTS="--ask --verbose --load-average --newuse --with-bdeps=y --keep-going --deep"
 CPU_FLAGS_X86="aes avx avx2 f16c fma3 mmx mmxext pclmul popcnt rdrand sse sse2 sse3 sse4_1 sse4_2 ssse3"
 AUTO_CLEAN="yes"

 PORTDIR="/var/db/repos/gentoo"
 DISTDIR="/var/cache/distfiles"
 PKGDIR="/var/chache/binpkgs"
 PORTAGE_TMPDIR="/tmp"

 PORTAGE_COMPRESS="zstd"
 BINPKG_COMPRESS="zstd"

 ACCEPT_LICENSE="*"
 ACCEPT_KEYWORDS="~amd64"
 GRUB_PLATFORMS="efi-64"
 VIDEO_CARDS="nouveau"

 GENTOO_MIRRORS="https://mirrors.tuna.tsinghua.edu.cn/gentoo"
 MICROCODE_SIGNATURES="-S"

 # NOTE: This stage was built with the bindist Use flag enabled

 # This sets the language of build output to English.
 # Please keep this setting intact when reporting bugs.
 LC_MESSAGES=C.utf8

 # ccache
 FEATURES="ccache -test"
 CCACHE_DIR="/var/cache/ccache"

 # aria2
 FETCHCOMMAND="/usr/bin/aria2c -d \${DISTDIR} -o \${FILE} --allow-overwrite=true --max-tries=5 --max-file-not-found=2 --max-concurrent-downloads=5 --connect-timeout=5 --timeout=5 --split=5 --min-split-size=2M --lowest-speed-limit=20K --max-connection-per-server=9 --uri-selector=feedback \${URI}"
 RESUMECOMMAND="${FETCHCOMMAND}"
```

### 问题列举

> 安装 Gentoo 是相对容易的，但是往往会遇上很多奇怪的问题。比如常见的循环依赖，某个程序编译失败...
> 这种时候首先应该去 Gentoo Package 查找对应包是否有 Bug 记录，以及解决方法
> 这里主要列举的是我在安装过程中遇到的问题，以及解决方法

#### 复杂密码换简单密码

- Gentoo 默认是复杂密码，为了便于日常使用，改为简单密码：

  ```passwdqc.conf
  /etc/security/passwdqc.conf

  min=8,8,8,8,8
  max=40
  passphrase=0
  match=4
  similar=permit
  random=24
  enforce=none
  retry=3
  ```

  或者全局 USE -pam，我还没有尝试

#### eudev 还是 systemd-utils？

Gentoo 目前用 systemd-utils 替代了原本的 eudev，所以解决办法有：

1. 使用 systemd-utils 管理设备，不要再安装 eudev，这是最简单的。官方的原话是：`in general, you should not worry about installing anything *udev manually by yourself and you should imho not have anything like that in your world file.`
2. 如果你痛恨和 systemd 有关的一切，可以为 systemd-utils 包-udev USE，然后应该就可以安装 eudev 了。需要注意的是，eudev 不应该被加入到 world file 中。此外还有一些其他的 USE 也会影响，总之这很麻烦。建议还是不要折腾了，systemd-utils 只是从 systemd 中分离出来的组件而已，它包含了 udev

#### Desktop profiles？

Desktop profiles 预设了很多 USE，并包含了一些 system 依赖。对于 KDE 和 GNOME 用户，Desktop profiles 中提供的增量可以省很多事。但是对于裸 WM 来说，没有必要为使用 Desktop，默认的 profiles 或者 systemd profiles 就可以了。在最小化的基础上，安装软件时检查 USE 并逐步添加自己的全局 USE

#### /tmp 挂载

有时候 fstab 中会忘记挂载/tmp，这样/tmp 目录在磁盘中，在 portage 运行时主要以/tmp 作为暂存目录，可能会反复读写 SSD 降低寿命。将/tmp 挂载到内存中，毕竟内存更加皮实耐用

```fstab
# size 的大小一般为内存大小的一半
tmpfs /tmp tmpfs rw,nosuid,noatime,nodev,size=16G,mode=1777 0 0
```

#### 循环依赖问题

是在安装 polybar 的时候遇到了这个问题，其他人反应 vim 也有这个循环依赖。貌似不是个别人遇到的问题
如下报错信息:

```

```

