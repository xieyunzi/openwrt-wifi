# mi wifi hd 安装 aria2

大概一两年前买的小米路由器, 一直只是普通使用, 最近实在有些不爽就动手准备自己安装一些软件

## 遇到的问题

### 问题1:

准备从官方仓库安装软件。

需要找到对应的处理器架构配置软件仓库地址, 一顿好找。思路是找到产品宣传相关的资料, 上面一般写了芯片信息, 由此 google。

mi wifi HD 路由器芯片 target/subtarget 是 ipq806x/generic 对应的 openwrt package architecture 是 arm_cortex-a15_neon-vfpv4

下面是配置好的信息

```
root@XiaoQiang:/data# cat /etc/opkg.conf
arch all 100
arch ipq806x 200
arch arm_cortex-a15_neon-vfpv4 300
arch any 600
arch noarch 700

dest root /data
dest ram /tmp
lists_dir ext /data/var/opkg-lists
option overlay_root /data


root@XiaoQiang:/data# cat /etc/opkg/distfeeds.conf
src/gz openwrt_core http://downloads.openwrt.org/releases/18.06.2/targets/ipq806x/generic/packages
src/gz openwrt_base http://downloads.openwrt.org/releases/packages-18.06/arm_cortex-a15_neon-vfpv4/base
src/gz openwrt_luci http://downloads.openwrt.org/releases/packages-18.06/arm_cortex-a15_neon-vfpv4/luci
src/gz openwrt_packages http://downloads.openwrt.org/releases/packages-18.06/arm_cortex-a15_neon-vfpv4/packages
src/gz openwrt_routing http://downloads.openwrt.org/releases/packages-18.06/arm_cortex-a15_neon-vfpv4/routing
src/gz openwrt_telephony http://downloads.openwrt.org/releases/packages-18.06/arm_cortex-a15_neon-vfpv4/telephony
```

### 问题2:

安装软件

```
opkg install aira2
```

安装官方仓库的包提示缺少 musl libc, `opkg list | grep libc` 没找到, 去网站找了一个 download 下来手动安装

```
wget http://archive.openwrt.org/snapshots/trunk/ipq806x/generic/packages/base/libc_1.1.16-1_ipq806x.ipk
opgk install libc_1.1.16-1_ipq806x.ipk
```

### 问题3:

系统版本老旧一些脚本有点小问题, 安装软件之后会提示 `default_postinst not found`

```
cat /data/usr/lib/opkg/info/aria2.postinst
#!/bin/sh
[ "${IPKG_NO_SCRIPT}" = "1" ] && exit 0
[ -x ${IPKG_INSTROOT}/lib/functions.sh ] || exit 0
. ${IPKG_INSTROOT}/lib/functions.sh
default_postinst $0 $@
```

看一下内容, 选择直接忽略 `export IPKG_NO_SCRIPT=1`

### 问题4:

miwifi 系统是 openwrt 修改的版本, 和主流版本最大的区别大概是 libc, 官方仓库使用的是 musl libc 而小米路由器使用的是 uClibc, 这两个库非二进制兼容, 所以安装官方仓库的软件会有问题。

路由器 `/lib` 目录是只读的, 无法把自己的库添加进去, 本来准备自己编译一些软件, 但感觉交叉编译太麻烦。

感谢 linux filesystem 优雅的设计, 可以直接 `mount` 目录用新的替换旧的

```
cp -r /lib /data/lib.copy
cp -r /data/lib/* /data/lib.copy/
cp -r /data/usr/lib/* /data/lib.copy/
mount /data/lib.copy/ /lib

export PATH=$PATH:/data/bin:/data/usr/bin
```

尝试执行一下 `aria2c -h`, it works

接下来就正常安装官方仓库里面的软件, ss (not iproute2) 之类都是有的

## references

- https://unix.stackexchange.com/questions/18061/why-does-sh-say-not-found-when-its-definitely-there
- https://openwrt.org/docs/guide-user/additional-software/opkg#configuration
- https://openwrt.org/toh/views/toh_admin_fw-pkg-download
- https://www.jianshu.com/p/1042483f90fe
- http://blog.makeex.com/2016/07/18/old-driver-take-you-fly-on-xiaomi-router/
