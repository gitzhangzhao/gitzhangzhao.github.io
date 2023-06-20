# Gentoo安装小记


### 前言
> 安装Gentoo并不复杂，很多人混淆了复杂和耗时。在安装的过程中，大部分的时间都在做别的事情。同时，Gentoo的安装步骤是清晰的，Handbook和各路神仙的总结实际上已经很全面了。因此，没有必要再做重复的劳动，一些个性化的关键点记录就足够了。

> 为了使系统保持Suckless，尽量避免用不到的功能，我还是继续沿用裸wm的方式。简而言之：gentoo + openrc + i3wm + polybar + nvim。我的目标是尽量在一天内完成系统的整体安装，再用一周时间进行小修小补。而在流程化的步骤下，一天内的实际安装时间在1小时左右，而其余时间都在做其他事情。

> 此外，对于小新pro这种散热垃圾的Laptop，一个带风扇的散热架是必要的。否则，`emerge -e @world` 是真的会卡死（哭）。

### 安装步骤
Gentoo的安装大体上是规范的，但是针对不同用户的需求和理念，也有不一样的方式。或多或少的，存在一些坑

我在安装过程主要参考的几个链接：

1. 官方Handbook，这是最权威的手册，当问题不确定时，以Handbook为准

[Handbook](https://wiki.gentoo.org/wiki/Handbook:AMD64/zh-cn)

2. 一篇较为详细的安装笔记，属于经验丰富的老玩家心得了，很有参考价值

[Gentoo安装流程分享(step by step)，第一篇之基本系统的安装](https://zhuanlan.zhihu.com/p/122222365)

3. OriPoin's blog，详细介绍了Gentoo的优化方式，但是没必要采用O3，会带来很多未知问题

[Emerge your world the lean way](https://blog.oripoin.me/2022/04/emerge-your-world-the-lean-way/)

[Optimize Your system the stupid way](https://blog.oripoin.me/2022/04/optimize-your-system-the-stupid-way/)

4. bitbili's blog，非常非常详细的介绍了Gentoo的安装和使用

[Gentoo Linux 安装及使用指南](https://bitbili.net/gentoo-linux-installation-and-usage-tutorial.html)

5. Yangmame的安装教程

[Gentoo安装教程](https://blog.yangmame.org/Gentoo%E5%AE%89%E8%A3%85%E6%95%99%E7%A8%8B.html)

6. ayamir的知乎记录

[2020-Gentoo双系统安装指北](https://zhuanlan.zhihu.com/p/166652475)

6. Google，Stack Overflow，gentoo wiki，arch wiki等

### make.conf
make.conf可以说是Gentoo的核心了，针对PC的配置、优化以及对系统的预期基本上都是在这个文件中定义的，首先列出我的：
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
- Gentoo默认是复杂密码，为了便于日常使用，改为简单密码：
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
  或者全局USE -pam，我还没有尝试

#### eudev还是systemd-utils？
Gentoo目前用systemd-utils替代了原本的eudev，所以解决办法有：
1. 使用systemd-utils管理设备，不要再安装eudev，这是最简单的。官方的原话是：`in general, you should not worry about installing anything *udev manually by yourself and you should imho not have anything like that in your world file.`
2. 如果你痛恨和systemd有关的一切，可以为systemd-utils包-udev USE，然后应该就可以安装eudev了。需要注意的是，eudev不应该被加入到world file中。此外还有一些其他的USE也会影响，总之这很麻烦。建议还是不要折腾了，systemd-utils只是从systemd中分离出来的组件而已，它包含了udev

#### Desktop profiles？
Desktop profiles预设了很多USE，并包含了一些system依赖。对于KDE和GNOME用户，Desktop profiles中提供的增量可以省很多事。但是对于裸WM来说，没有必要为使用Desktop，默认的profiles或者systemd profiles就可以了。在最小化的基础上，安装软件时检查USE并逐步添加自己的全局USE


