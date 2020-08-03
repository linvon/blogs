---
title: "OpenWrt升级脚本sysupgrade详解"
date: "2019-05-16"
tags: ["openwrt"]
categories: ["学习笔记", "openwrt"]
---
以[LEDE-17.01版本](https://github.com/openwrt/openwrt/blob/lede-17.01/package/base-files/files/sbin/sysupgrade)为例，部分内容参考[Here](https://www.cnblogs.com/sammei/p/3973322.html)

> 首先是导出默认的选项为环境变量，提供给后续过程

``` bash
# initialize defaults
RAMFS_COPY_BIN=""	    # 临时文件系统的执行程序
RAMFS_COPY_DATA=""	    # 临时文件系统的数据文件
export MTD_CONFIG_ARGS=""
export INTERACTIVE=0    # 交互模式
export VERBOSE=1        # 输出更多信息
export SAVE_CONFIG=1    # 保留配置
export SAVE_OVERLAY=0   # 保留/etc目录修改过的文件
export SAVE_PARTITIONS=1
export DELAY=           # 重启前等待时间
export CONF_IMAGE=      # 恢复配置使用的tar.gz压缩包
export CONF_BACKUP_LIST=0 # 配置保留列表
export CONF_BACKUP=     # 保留配置的压缩包名称
export CONF_RESTORE=    # 恢复配置使用的压缩包名称
export NEED_IMAGE=      # 是否需要镜像
export HELP=0           # 输出帮助
export FORCE=0          # 强制升级
export TEST=0           # 测试镜像
```

> 参数解析

``` bash
# parse options
while [ -n "$1" ]; do
case "$1" in
-i) export INTERACTIVE=1;;
-d) export DELAY="$2"; shift;;
-v) export VERBOSE="$(($VERBOSE + 1))";;
-q) export VERBOSE="$(($VERBOSE - 1))";;
-n) export SAVE_CONFIG=0;;
-c) export SAVE_OVERLAY=1;;
-p) export SAVE_PARTITIONS=0;;
-b|--create-backup) export CONF_BACKUP="$2" NEED_IMAGE=1; shift;;
-r|--restore-backup) export CONF_RESTORE="$2" NEED_IMAGE=1; shift;;
-l|--list-backup) export CONF_BACKUP_LIST=1; break;;
-f) export CONF_IMAGE="$2"; shift;;
-F|--force) export FORCE=1;;
-T|--test) export TEST=1;;
-h|--help) export HELP=1; break;;
-*)
echo "Invalid option: $1"
exit 1
;;
*) break;;
esac
shift;
done
```

- 升级选项：
  -d 重启前等待 delay 秒  
  -f 从 .tar.gz (文件或链接) 中恢复配置文件  
  -i 交互模式  
  -c 保留 /etc 中所有修改过的文件  
  -n 重刷固件时不保留配置文件  
  -T | --test 校验固件 config .tar.gz，但不真正烧写  
  -F | --force 即使固件校验失败也强制烧写  
  -q 较少的输出信息  
  -v 详细的输出信息  
  -h 显示帮助信息  
  备份选项：  
  -b | --create-backup   
  把sysupgrade.conf 里描述的文件打包成.tar.gz 作为备份，不做烧写动作  
  -r | --restore-backup   
  从-b 命令创建的 .tar.gz 文件里恢复配置，不做烧写动作  
  -l | --list-backup  
  列出 -b 命令将备份的文件列表，但不创建备份文件  

> 导出环境变量

``` bash
export CONFFILES=/tmp/sysupgrade.conffiles
export CONF_TAR=/tmp/sysupgrade.tgz

export ARGV="$*"
export ARGC="$#"
```
CONFFILES和CONF_TAR为临时文件  
CONFFILES表示配置文件列表，CONF_TAR表示配置压缩包  
ARGV表示参数，ARGC表示参数个数  
`sysupgrade openwrt.bin` --> ARGV="openwrt.bin", ARGC=1  
`sysupgrade -b config.backup` --> ARGV为空，ARGC=0  

> 检查参数

``` bash
[ -z "$ARGV" -a -z "$NEED_IMAGE" -o $HELP -gt 0 ] && {
				cat <<EOF
				..........
				EOF
				exit 1
}
```

如果ARGV是空的并且没有附带参数-b或-r，或者带了-h参数时，输出帮助信息  

> 对-b,-r参数做校验

``` bash
[ -n "$ARGV" -a -n "$NEED_IMAGE" ] && {
	cat <<-EOF
		-b|--create-backup and -r|--restore-backup do not perform a firmware upgrade.
		Do not specify both -b|-r and a firmware image.
	EOF
	exit 1
}
```

如果sysupgrade附带参数-b或-r时，则$NEED_IMAGE=1，否则为空  
当$NEED_IMAGE=1时，我们希望ARGV是空的，否则就是出错，则输出提示信息，并退出。  

> 备份配置时关闭详细输出

``` bash
# prevent messages from clobbering the tarball when using stdout
[ "$CONF_BACKUP" = "-" ] && export VERBOSE=0
```

> 列出opkg维护的配置文件列表

``` bash
list_conffiles() {
	awk '
		BEGIN { conffiles = 0 }
		/^Conffiles:/ { conffiles = 1; next }
		!/^ / { conffiles = 0; next }
		conffiles == 1 { print }
	' /usr/lib/opkg/status
}
```

> 列出opkg维护的配置列表中有改动的文件列表

``` bash
list_changed_conffiles() {
	# Cannot handle spaces in filenames - but opkg cannot either...
	list_conffiles | while read file csum; do
		[ -r "$file" ] || continue

		echo "${csum}  ${file}" | sha256sum -sc - || echo "$file"
	done
}
```

> 列出要保留的配置文件列表

``` bash
add_uci_conffiles() {
	local file="$1"
	( find $(sed -ne '/^[[:space:]]*$/d; /^#/d; p' \
		/etc/sysupgrade.conf /lib/upgrade/keep.d/* 2>/dev/null) \
		-type f -o -type l 2>/dev/null;
	  list_changed_conffiles ) | sort -u > "$file"
	return 0
}
```

从`/etc/sysupgrade.conf` `/lib/upgrade/keep.d/*` `list_changed_conffiles`中获取保留列表，写入文件

> 获取/etc下修改过的文件列表，写入文件

``` bash
add_overlayfiles() {
	local file="$1"
	if [ -d /overlay/upper ]; then
		local overlaydir="/overlay/upper"
	else
		local overlaydir="/overlay"
	fi
	find $overlaydir/etc/ -type f -o -type l | sed \
		-e 's,^/overlay\/upper/,/,' \
		-e 's,^/overlay/,/,' \
		-e '\,/META_[a-zA-Z0-9]*$,d' \
		-e '\,/functions.sh$,d' \
		-e '\,/[^/]*-opkg$,d' \
	> "$file"
	return 0
}
```

由于OpenWrt的特殊挂载模式，查找/overlay/upper/etc目录，即可获得修改过的文件

> 创建初始hook

``` bash
# hooks
sysupgrade_image_check="fwtool_check_image platform_check_image"
sysupgrade_pre_upgrade="fwtool_pre_upgrade"
[ $SAVE_OVERLAY = 0 -o ! -d /overlay/etc ] && \
	sysupgrade_init_conffiles="add_uci_conffiles" || \
	sysupgrade_init_conffiles="add_overlayfiles"
```

run hook时即会执行hook中记录的函数  
`sysupgrade_image_check`为执行镜像检查  
`sysupgrade_pre_upgrade`为预处理升级参数  
`sysupgrade_init_conffiles`为处理保留配置：不带-c参数（-c:保留/etc所有修改过的文件）或者`/overlay/etc`目录不存在,便只保留配置列表中的文件，否则保留`/etc`所有修改过的文件

> 加载相关脚本

``` bash
include /lib/upgrade
```

> 指定nand时，则调用nand_upgrade_stage2函数（基本使用flash，不关心这里）

``` bash
[ "$1" = "nand" ] && nand_upgrade_stage2 $@
```

> 获取保留列表，将相关文件压缩到指定压缩包

``` bash
do_save_conffiles() {
	local conf_tar="${1:-$CONF_TAR}"

	[ -z "$(rootfs_type)" ] && {
		echo "Cannot save config while running from ramdisk."
		ask_bool 0 "Abort" && exit
		return 0
	}
	run_hooks "$CONFFILES" $sysupgrade_init_conffiles
	ask_bool 0 "Edit config file list" && vi "$CONFFILES"

	v "Saving config files..."
	[ "$VERBOSE" -gt 1 ] && TAR_V="v" || TAR_V=""
	tar c${TAR_V}zf "$conf_tar" -T "$CONFFILES" 2>/dev/null

	rm -f "$CONFFILES"
}
```

> 判断带-l参数则输出保留配置列表

``` bash
if [ $CONF_BACKUP_LIST -eq 1 ]; then
	add_uci_conffiles "$CONFFILES"
	cat "$CONFFILES"
	rm -f "$CONFFILES"
	exit 0
fi
```

获取保留列表到临时文件，显示并删除临时文件，退出脚本

> 判断带-b参数且指定备份名称则备份配置文件到指定文件

``` bash
if [ -n "$CONF_BACKUP" ]; then
	do_save_conffiles "$CONF_BACKUP"
	exit $?
fi
```

> 判断带-r参数且指定了备份名称，则用指定的配置恢复备份

``` bash
if [ -n "$CONF_RESTORE" ]; then
	if [ "$CONF_RESTORE" != "-" ] && [ ! -f "$CONF_RESTORE" ]; then
		echo "Backup archive '$CONF_RESTORE' not found."
		exit 1
	fi

	[ "$VERBOSE" -gt 1 ] && TAR_V="v" || TAR_V=""
	tar -C / -x${TAR_V}zf "$CONF_RESTORE"
	exit $?
fi

```

> 检查镜像

``` bash
type platform_check_image >/dev/null 2>/dev/null || {
	echo "Firmware upgrade is not implemented for this platform."
	exit 1
}

for check in $sysupgrade_image_check; do
	( eval "$check \"\$ARGV\"" ) || {
		if [ $FORCE -eq 1 ]; then
			echo "Image check '$check' failed but --force given - will update anyway!"
			break
		else
			echo "Image check '$check' failed."
			exit 1
		fi
	}
done
```

执行hook $sysupgrade_image_check：  
1. fwtool_check_image

``` bash
fwtool_check_image() {
	[ $# -gt 1 ] && return 1

	. /usr/share/libubox/jshn.sh

	if ! fwtool -q -i /tmp/sysupgrade.meta "$1"; then  # 从升级包解压出包信息，即支持升级的硬件
		echo "Image metadata not found"
		[ "$REQUIRE_IMAGE_METADATA" = 1 -a "$FORCE" != 1 ] && {
			echo "Use sysupgrade -F to override this check when downgrading or flashing to vendor firmware"
		}
		[ "$REQUIRE_IMAGE_METADATA" = 1 ] && return 1
		return 0
	fi

	json_load "$(cat /tmp/sysupgrade.meta)" || {
		echo "Invalid image metadata"
		return 1
	}

	device="$(cat /tmp/sysinfo/board_name)"

	json_select supported_devices || return 1

  json_get_keys dev_keys  # 对升级包信息与设备信息进行匹配校验
  for k in $dev_keys; do
  	json_get_var dev "$k"
  	[ "$dev" = "$device" ] && return 0
  done

  echo "Device $device not supported by this image"
  echo -n "Supported devices:"
  for k in $dev_keys; do
  	json_get_var dev "$k"
  	echo -n " $dev"
  done
  echo

  return 1
}
```

2. platform_check_image

> 处理配置保留

``` bash
if [ -n "$CONF_IMAGE" ]; then  # 带-f参数则从tar.gz包中载入配置
	case "$(get_magic_word $CONF_IMAGE cat)" in
		# .gz files
		1f8b) ;;
		*)
			echo "Invalid config file. Please use only .tar.gz files"
			exit 1
		;;
	esac
	get_image "$CONF_IMAGE" "cat" > "$CONF_TAR" # 从压缩包中16进制取内容，存到CONF_TAR
	export SAVE_CONFIG=1
elif ask_bool $SAVE_CONFIG "Keep config files over reflash"; then # 带-n参数则不保存配置（默认不保存）
	[ $TEST -eq 1 ] || do_save_conffiles  # 带-T参数，则不生成配置压缩包
	export SAVE_CONFIG=1
else
	export SAVE_CONFIG=0
fi
```

> 退出测试升级

``` bash
if [ $TEST -eq 1 ]; then
	exit 0
fi
```

> 执行hook，进行预处理

``` bash
run_hooks "" $sysupgrade_pre_upgrade
```

> 如果定义了特殊升级方式，在此执行

``` bash
if type 'platform_pre_upgrade' >/dev/null 2>/dev/null; then
	platform_pre_upgrade "$ARGV"
fi
```

> 通知系统升级，创建升级标记

``` bash
ubus call system upgrade
touch /tmp/sysupgrade
```

> 判断设备是否在安全模式，如果不在就开始关闭进程

``` bash
if [ ! -f /tmp/failsafe ] ; then
	kill_remaining TERM
sleep 3
	kill_remaining KILL
fi
```

> 判断数据分区的挂载，如果挂载在/overlay就切换到内存临时文件系统，执行升级

``` bash
if [ -n "$(rootfs_type)" ]; then
v "Switching to ramdisk..."
	run_ramfs '. /lib/functions.sh; include /lib/upgrade; do_upgrade'
else
	do_upgrade
fi
```

run_ramfs会在/tmp/root下安装一个临时内存文件系统，最后再执行do_upgrade

``` bash
run_ramfs() { # <command> [...]
  install_bin ....    # 已省略，安装诸多bin工具
  for file in $RAMFS_COPY_BIN; do
  	install_bin ${file//:/ }
  done
  install_file /etc/resolv.conf /lib/*.sh /lib/functions/*.sh /lib/upgrade/*.sh $RAMFS_COPY_DATA

  [ -L "/lib64" ] && ln -s /lib $RAM_ROOT/lib64

  supivot $RAM_ROOT /mnt || {   # 把根目录都转移挂载到内存上
  	echo "Failed to switch over to ramfs. Please reboot."
  	exit 1
  }

  /bin/mount -o remount,ro /mnt  # 将原挂载点挂为只读
  /bin/umount -l /mnt            # 安全卸载原挂载点

  grep /overlay /proc/mounts > /dev/null && {   # 如果overlay还在挂载，将其挂为只读并安全写在
  	/bin/mount -o noatime,remount,ro /overlay
  	/bin/umount -l /overlay
  }

  # spawn a new shell from ramdisk to reduce the probability of cache issues
  exec /bin/busybox ash -c "$*"    # 使用ash执行传入的命令
}
```

``` bash
# Flash firmware to MTD partition
#
# $(1): path to image
# $(2): (optional) pipe command to extract firmware, e.g. dd bs=n skip=m
default_do_upgrade() {
  sync
  if [ "$SAVE_CONFIG" -eq 1 ]; then # 判断是否保留配置，并将镜像包（和配置）数据获取并写入flash
  	get_image "$1" "$2" | mtd $MTD_CONFIG_ARGS -j "$CONF_TAR" write - "${PART_NAME:-image}"
  else
  	get_image "$1" "$2" | mtd write - "${PART_NAME:-image}"
  fi
}

do_upgrade() {
  v "Performing system upgrade..."
  if type 'platform_do_upgrade' >/dev/null 2>/dev/null; then # 特殊平台升级单独处理
  	platform_do_upgrade "$ARGV"
  else
  	default_do_upgrade "$ARGV"
  fi

  if [ "$SAVE_CONFIG" -eq 1 ] && type 'platform_copy_config' >/dev/null 2>/dev/null; then
  	platform_copy_config
  fi

  v "Upgrade completed"
  [ -n "$DELAY" ] && sleep "$DELAY"
  ask_bool 1 "Reboot" && {     #  升级完成 重启
  	v "Rebooting system..."
    umount -a
  	reboot -f
  	sleep 5
  	echo b 2>/dev/null >/proc/sysrq-trigger
  }
}
```