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

# build-erlang-for-android
Build Erlang For Android

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
