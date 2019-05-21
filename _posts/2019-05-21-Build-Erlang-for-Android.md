---
layout:     post              # 使用的布局（不需要改）
title:      Build Erlang for Android        # 标题 
date:       2019-05-21        # 时间
author:     Chuan            # 作者
header-img: img/post-bg-2015.jpg  #这篇文章标题背景图片
catalog: true             # 是否归档
tags:               #标签
    - Erlang
    - Android
---

# 在Android上跑命令行程序的常规方法:

  1. 交叉编译
  2. adb push /data/local/tmp/somedir
  3. adb shell
  4. cd /data/local/tmp/somedir
  5. run your bin


但，上述方法对Erlang有一定限制:
`crypto:start().`
执行失败。

具体原因: android7.0之后对加载so做了白名单限制。
`crypto:start().`
方法会通过dlopen加载"$ROOTDIR/lib/crypto-4.3.3/priv/lib/crypto.so"，dlopen因为白名单限制，
执行失败。


# 绕过该限制的方法：

  - 修改rom
  - 开发一个apk壳 

修改rom的方法通用型差。

方法2，具体实现:
  1. 开发一个apk壳
  2. 把Erlang整个环境放到assets中
  3. apk运行时，把Erlang部署到 "/data/data/com.your-apk-packagename/erlang"下
  4. adb shell
  5. run-as com.your-apk-packagename
  6. cd erlang
  7. sh bin/erl

制作apk壳的方法之所以能绕过限制，是因为：linker创建了属于app本身的namespace，保证app可以加载本沙盒内的so库，
且不会对其他程序造成影响。这是Android独有的linker行为，给熟悉Linux开发的朋友挖了个大坑；Android发展太快，
版本太多，bless那些有native开发需求的朋友。


# Tools and Code Info

- Ubuntu 16.04.2
- android ndk r18b
- openssl-1.1.0f
- otp_src_21.1
- 小米MIX2 (MIUI9.2, Android7.1.1)


# NDK toolchain gen(64)

查看目标平台api信息

`adb shell getprop ro.build.version.sdk` 


生成toolchain

`$NDK/build/tools/make_standalone_toolchain.py --arch arm64 --api 24 --install-dir /tmp/toolchain64_24`


# OpenSSL 编译(64)

`bash ./build-openssl4android.sh android64-aarch64`

参考: 
https://github.com/leenjewel/openssl_for_ios_and_android.git

# Erlang 编译(64)

把erl-xcomp-arm64-android.conf置于xcomp目录

1. setup x-compile env
2. ./opt_build autoconf
3. ./opt_build configure --xcomp-conf=xcomp/erl-xcomp-arm64-android.conf
4. ./opt_build boot -a

参考: 
https://github.com/erlang/otp/blob/master/HOWTO/INSTALL-ANDROID.md
https://bluishcoder.co.nz/2015/06/21/building-erlang-for-android.html


# android 7.0后对加载so的限制

https://developer.android.com/about/versions/nougat/android-7.0-changes?hl=zh-cn#ndk

https://android.googlesource.com/platform/bionic/+/master/android-changes-for-ndk-developers.md

https://source.android.google.cn/devices/architecture/vndk/linker-namespace

http://jackwish.net/namespace-based-dynamic-linking.html

https://www.jianshu.com/p/4be3d1dafbec


# linux gen core dump

  - gcc -g才能产生core
  - ulimit -c unlimited


# 验证Erlang crypto模块

http://marianoguerra.org/posts/publicprivate-key-encryption-sign-and-verification-in-erlang.html  

# erl-xcomp-arm64-android.conf

```
## -*-shell-script-*-
##
## %CopyrightBegin%
##
## Copyright Ericsson AB 2009-2010. All Rights Reserved.
##
## Licensed under the Apache License, Version 2.0 (the "License");
## you may not use this file except in compliance with the License.
## You may obtain a copy of the License at
##
##     http://www.apache.org/licenses/LICENSE-2.0
##
## Unless required by applicable law or agreed to in writing, software
## distributed under the License is distributed on an "AS IS" BASIS,
## WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
## See the License for the specific language governing permissions and
## limitations under the License.
##
## %CopyrightEnd%
##
## File: erl-xcomp.conf.template
## Author:
##
## -----------------------------------------------------------------------------
## When cross compiling Erlang/OTP using `otp_build', copy this file and set
## the variables needed below. Then pass the path to the copy of this file as
## an argument to `otp_build' in the configure stage:
##   `otp_build configure --xcomp-conf=<FILE>'
## -----------------------------------------------------------------------------

## Note that you cannot define arbitrary variables in a cross compilation
## configuration file. Only the ones listed below will be guaranteed to be
## visible throughout the whole execution of all `configure' scripts. Other
## variables needs to be defined as arguments to `configure' or exported in
## the environment.

## -- Variables for `otp_build' Only -------------------------------------------

## Variables in this section are only used, when configuring Erlang/OTP for
## cross compilation using `$ERL_TOP/otp_build configure'.

## *NOTE*! These variables currently have *no* effect if you configure using
## the `configure' script directly.

# * `erl_xcomp_build' - The build system used. This value will be passed as
#   `--build=$erl_xcomp_build' argument to the `configure' script. It does
#   not have to be a full `CPU-VENDOR-OS' triplet, but can be. The full
#   `CPU-VENDOR-OS' triplet will be created by
#   `$ERL_TOP/erts/autoconf/config.sub $erl_xcomp_build'. If set to `guess',
#   the build system will be guessed using
#   `$ERL_TOP/erts/autoconf/config.guess'.
erl_xcomp_build=guess

# * `erl_xcomp_host' - Cross host/target system to build for. This value will
#   be passed as `--host=$erl_xcomp_host' argument to the `configure' script.
#   It does not have to be a full `CPU-VENDOR-OS' triplet, but can be. The
#   full `CPU-VENDOR-OS' triplet will be created by
#   `$ERL_TOP/erts/autoconf/config.sub $erl_xcomp_host'.
erl_xcomp_host=aarch64-linux-android

# * `erl_xcomp_configure_flags' - Extra configure flags to pass to the
#   `configure' script.
erl_xcomp_configure_flags="--disable-hipe --without-termcap --with-ssl=$OPENSSL --enable-dynamic-ssl-lib=no --without-wx --enable-m64-build"


## -- Cross Compiler and Other Tools -------------------------------------------

##
##
NDK_SYSROOT=$NDK_ROOT/sysroot

## If the cross compilation tools are prefixed by `<HOST>-' you probably do
## not need to set these variables (where `<HOST>' is what has been passed as
## `--host=<HOST>' argument to `configure').

## All variables in this section can also be used when native compiling.

# * `CC' - C compiler.
#aarch64-linux-android-gcc
CC="aarch64-linux-android-gcc --sysroot=$NDK_SYSROOT"

# * `CFLAGS' - C compiler flags.
CFLAGS="-target aarch64-linux-android -pie -fPIE"

# * `STATIC_CFLAGS' - Static C compiler flags.
#STATIC_CFLAGS=

# * `CFLAG_RUNTIME_LIBRARY_PATH' - This flag should set runtime library
#   search path for the shared libraries. Note that this actually is a
#   linker flag, but it needs to be passed via the compiler.
#CFLAG_RUNTIME_LIBRARY_PATH=

# * `CPP' - C pre-processor.
CPP="aarch64-linux-android-gcc -E --sysroot=$NDK_SYSROOT"

# * `CPPFLAGS' - C pre-processor flags.
CPPFLAGS="-target aarch64-linux-android"

# * `CXX' - C++ compiler.
CXX="aarch64-linux-android-g++ --sysroot=$NDK_SYSROOT"

# * `CXXFLAGS' - C++ compiler flags.
CXXFLAGS="-taget aarch64-linux-android"

# * `LD' - Linker.
LD="aarch64-linux-anroid-ld"

# * `LDFLAGS' - Linker flags.
LDFLAGS="-target aarch64-linux-android -Wl,--dynamic-linker=/system/bin/linker64 -pie -fPIE"

# * `LIBS' - Libraries.
LIBS="-ldl -lz"

## -- *D*ynamic *E*rlang *D*river Linking --

## *NOTE*! Either set all or none of the `DED_LD*' variables.

# * `DED_LD' - Linker for Dynamically loaded Erlang Drivers.
#DED_LD=

# * `DED_LDFLAGS' - Linker flags to use with `DED_LD'.
#DED_LDFLAGS=

# * `DED_LD_FLAG_RUNTIME_LIBRARY_PATH' - This flag should set runtime library
#   search path for shared libraries when linking with `DED_LD'.
#DED_LD_FLAG_RUNTIME_LIBRARY_PATH=

## -- Large File Support --

## *NOTE*! Either set all or none of the `LFS_*' variables.

# * `LFS_CFLAGS' - Large file support C compiler flags.
#LFS_CFLAGS=

# * `LFS_LDFLAGS' - Large file support linker flags.
#LFS_LDFLAGS=

# * `LFS_LIBS' - Large file support libraries.
#LFS_LIBS=

## -- Other Tools --

# * `RANLIB' - `ranlib' archive index tool.
#RANLIB=

# * `AR' - `ar' archiving tool.
#AR=

# * `GETCONF' - `getconf' system configuration inspection tool. `getconf' is
#   currently used for finding out large file support flags to use, and
#   on Linux systems for finding out if we have an NPTL thread library or
#   not.
#GETCONF=

## -- Cross System Root Locations ----------------------------------------------

# * `erl_xcomp_sysroot' - The absolute path to the system root of the cross
#   compilation environment. Currently, the `crypto', `odbc', `ssh' and
#   `ssl' applications need the system root. These applications will be
#   skipped if the system root has not been set. The system root might be
#   needed for other things too. If this is the case and the system root
#   has not been set, `configure' will fail and request you to set it.
erl_xcomp_sysroot="$NDK_SYSROOT"


# * `erl_xcomp_isysroot' - The absolute path to the system root for includes
#   of the cross compilation environment. If not set, this value defaults
#   to `$erl_xcomp_sysroot', i.e., only set this value if the include system
#   root path is not the same as the system root path.
#erl_xcomp_isysroot=

## -- Optional Feature, and Bug Tests ------------------------------------------

## These tests cannot (always) be done automatically when cross compiling. You
## usually do not need to set these variables. Only set these if you really
## know what you are doing.

## Note that some of these values will override results of tests performed
## by `configure', and some will not be used until `configure' is sure that
## it cannot figure the result out.

## The `configure' script will issue a warning when a default value is used.
## When a variable has been set, no warning will be issued.

# * `erl_xcomp_after_morecore_hook' - `yes|no'. Defaults to `no'. If `yes',
#   the target system must have a working `__after_morecore_hook' that can be
#   used for tracking used `malloc()' implementations core memory usage.
#   This is currently only used by unsupported features.
#erl_xcomp_after_morecore_hook=

# * `erl_xcomp_bigendian' - `yes|no'. No default. If `yes', the target system
#   must be big endian. If `no', little endian. This can often be
#   automatically detected, but not always. If not automatically detected,
#   `configure' will fail unless this variable is set. Since no default
#   value is used, `configure' will try to figure this out automatically.
#erl_xcomp_bigendian=

# * `erl_xcomp_clock_gettime_cpu_time' - `yes|no'. Defaults to `no'. If `yes',
#   the target system must have a working `clock_gettime()' implementation
#   that can be used for retrieving process CPU time.
#erl_xcomp_clock_gettime_cpu_time=

# * `erl_xcomp_getaddrinfo' - `yes|no'. Defaults to `no'. If `yes', the target
#   system must have a working `getaddrinfo()' implementation that can
#   handle both IPv4 and IPv6.
#erl_xcomp_getaddrinfo=

# * `erl_xcomp_gethrvtime_procfs_ioctl' - `yes|no'. Defaults to `no'. If `yes',
#   the target system must have a working `gethrvtime()' implementation and
#   is used with procfs `ioctl()'.
#erl_xcomp_gethrvtime_procfs_ioctl=

# * `erl_xcomp_dlsym_brk_wrappers' - `yes|no'. Defaults to `no'. If `yes', the
#   target system must have a working `dlsym(RTLD_NEXT, <S>)' implementation
#   that can be used on `brk' and `sbrk' symbols used by the `malloc()'
#   implementation in use, and by this track the `malloc()' implementations
#   core memory usage. This is currently only used by unsupported features.
#erl_xcomp_dlsym_brk_wrappers=

# * `erl_xcomp_kqueue' - `yes|no'. Defaults to `no'. If `yes', the target
#   system must have a working `kqueue()' implementation that returns a file
#   descriptor which can be used by `poll()' and/or `select()'. If `no' and
#   the target system has not got `epoll()' or `/dev/poll', the kernel-poll
#   feature will be disabled.
#erl_xcomp_kqueue=

# * `erl_xcomp_linux_clock_gettime_correction' - `yes|no'. Defaults to `yes' on
#   Linux; otherwise, `no'. If `yes', `clock_gettime(CLOCK_MONOTONIC, _)' on
#   the target system must work. This variable is recommended to be set to
#   `no' on Linux systems with kernel versions less than 2.6.
#erl_xcomp_linux_clock_gettime_correction=

# * `erl_xcomp_linux_nptl' - `yes|no'. Defaults to `yes' on Linux; otherwise,
#   `no'. If `yes', the target system must have NPTL (Native POSIX Thread
#   Library). Older Linux systems have LinuxThreads instead of NPTL (Linux
#   kernel versions typically less than 2.6).
#erl_xcomp_linux_nptl=

# * `erl_xcomp_linux_usable_sigaltstack' - `yes|no'. Defaults to `yes' on Linux;
#   otherwise, `no'. If `yes', `sigaltstack()' must be usable on the target
#   system. `sigaltstack()' on Linux kernel versions less than 2.4 are
#   broken.
#erl_xcomp_linux_usable_sigaltstack=

# * `erl_xcomp_linux_usable_sigusrx' - `yes|no'. Defaults to `yes'. If `yes',
#   the `SIGUSR1' and `SIGUSR2' signals must be usable by the ERTS. Old
#   LinuxThreads thread libraries (Linux kernel versions typically less than
#   2.2) used these signals and made them unusable by the ERTS.
#erl_xcomp_linux_usable_sigusrx=

# * `erl_xcomp_poll' - `yes|no'. Defaults to `no' on Darwin/MacOSX; otherwise,
#   `yes'. If `yes', the target system must have a working `poll()'
#   implementation that also can handle devices. If `no', `select()' will be
#   used instead of `poll()'.
#erl_xcomp_poll=

# * `erl_xcomp_putenv_copy' - `yes|no'. Defaults to `no'. If `yes', the target
#   system must have a `putenv()' implementation that stores a copy of the
#   key/value pair.
#erl_xcomp_putenv_copy=

# * `erl_xcomp_reliable_fpe' - `yes|no'. Defaults to `no'. If `yes', the target
#   system must have reliable floating point exceptions.
#erl_xcomp_reliable_fpe=

## -----------------------------------------------------------------------------
```
