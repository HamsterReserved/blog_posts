---
title: What I've Done for Android Q port on Pixel C
date: 2019-03-24 01:35:35
tags: [Android]
---

（恰好在听林肯公园的 Minutes to Midnight 这张专辑，于是标题就这么来了）

以前一直想记录一次新设备上新版本 Android 时源码移植的经过，但是从源码编译时经常是对着日志一阵乱改就好了，过于爽快，没有时间记录中间遇到的问题，能在最终的提交信息里解释清楚为什么要这么改已经不太容易了 [doge] 当然，懒是更重要的原因。

这次因为没有源码[1]，只能拿 Pixel 系列现有的镜像拼包，没有版本控制，啥更改都要自己记[2]，所以有条件在每次出问题后整理整个问题的来龙去脉了。现在把拼包过程记录一下，算是完成了前述想法的一部分吧。

[1] 之前每次预览版也都跟这次一样会放出一个 tag，但以前很多仓库里这个 tag 上都没有新代码。不过这次似乎不一样，很多库都放出了真正的源码——我也没去逐个看就是了，本来工作以后时间就巨特么少，不想承担代码不完整带来的时间浪费的风险。反正梯子也突然不稳……

[2] 即便是新 iPad air 和 iPad mini 都支持 Apple Pencil 了，我还是要说 iPad Pro 10.5 (2017) 真香！

## 基础镜像准备

### 信息收集

之前就听说过 Pixel Q 的镜像是 System as Root（下称 SAR）的，所以要给 Pixel C 编译出一份能用的开启了 SAR 的 Android P 镜像打底。

而又听说过 SAR 要求 `BOARD_VNDK_VERSION := current`，所以要开着这个选项编译上面 SAR 的镜像。

为了避免步子太大扯到蛋，所以先只加这一点修改：Treble 化。SAR 待这步完成再说。

既然  `BOARD_VNDK_VERSION := current` 了，那顺便也 `PRODUCT_FULL_TREBLE_OVERRIDE := true` 吧，不然也太不值了。

既然准备要 Treble 了，干脆把 device tree 里新编译的东西都丢进 vendor 好了，加一堆的 `LOCAL_VENDOR_MODULE := true` 尽量避免依赖 system.

其他在 device tree 需要添加的 flag 有：

```makefile
PRODUCT_VENDOR_MOVE_ENABLED := true

PRODUCT_PACKAGES += \
    vndk-sp

PRODUCT_SHIPPING_API_LEVEL := 23
```

内核方面需要在 DTS 里添加 system 和 vendor 的 early mount. 这点做起来又简单又有成就感，性价比非常高，可惜我在 Pixel C 刚上魔趣的第二天就做过了（为了解决 system 里的 rc 文件中早于挂载文件系统的 trigger 失效的问题，会导致 WebView 白屏，详情自己看 log）

### 编译 Treble 开启的镜像

于是先带着上述更改跑了编译，出现了成吨的错误

#### `pthread_create()` 等常见到不能再常见的标准库函数定义找不到

因为在开启 VNDK 后 Google 移除了不少预包含的头文件，所以用到啥就得自己去把它包含进来，比如 pthread.h. 讲道理我觉得这是个基本的编程习惯啊……

> Pixel C 上这一步做起来还挺容易的，大部分出现这个问题的代码都在 device tree 里面，改得不多。OnePlus 5 改得那叫一个想骂人，HAL 里面这一个那一个的。

#### libhardware/xxx.h 等 Android 专属的头文件找不到

因为在开启 VNDK 后 Google 移除了不少预定义的头文件路径，所以用到啥就得自己去把它包含进来。这里有两种方式：

   1. 一般这样被其他模块包含的头文件都会属于某个 header library，在模块的编译配置文件中引用此 library 即可。比如上面举例的 libhardware/xxx.h 就属于 libhardware_headers 这个 header library，需在出错文件的 Android.mk 里添加 `LOCAL_HEADER_LIBRARIES :=  libhardware_headers`，Android.bp 则是 `header_libs: ["libhardware_headers"]`。

      > 怎么找到一个头文件是哪个 header library 的呢？找到它本体所在的目录，看它的 Android.mk/bp 中是否有 `include $(BUILD_HEADER_LIBRARY)` `cc_library_headers` 这类语句/定义，照着 name 抄下来就是了。那么怎么找到它所在的目录呢？
      >
      > 1. 凭经验和直觉
      > 2. 凭 IOPS 爆炸的 SSD

   2. 如果找不到包含这个头文件的 header library，就只能回归传统，用 `LOCAL_C_INCLUDES += ...` 或者 `include_dirs := ["..."]` 实现了。不过好像只有高通的一些 HAL 会需要这么做，Pixel C 只要添加各种 header library 就好了。

> 这个问题因为改了 Android.mk/bp，所以每次测试都会重新 include 一遍整个源码树的编译配置文件，很磨耐心。Pixel C 的情况与上一条相似，OnePlus 5 就哈哈哈哈哈哈哈哈哈你猜我花了多久才验证完

#### SEPolicy 花式违反 neverallow 规则

注意，由于有些问题我没有投入时间去寻找最佳解法，有的根本就没法解，所以这里优先保证 SEPolicy 编译通过，其次才是保持正确。建议 permissive 吧……

Pixel C 上出现过的几种情况：

   1. 规则限定范围太大

      如果确认这个对象需要访问的内容符合 neverallow 规则，只需要把违反规则的部分剔除掉即可。例如下例中我可以确认 cameraserver 不需访问 vendor app / overlay 类别的文件，则：

      ```diff
      -allow cameraserver vendor_file_type:dir r_dir_perms;
      +allow cameraserver {
      +    vendor_file_type
      +    -vendor_app_file
      +    -vendor_overlay_file # 这两行是直接从 system 里面的 neverallow 规则抄过来的
      +}:dir r_dir_perms;
      ```

   2. 文件移到 vendor 后不允许与 system 产生太多关系

      例如 vendor 下面的 shell 文件就不能由 system 下面的 sh 来解释。shebang 记得一起改了。

      ```diff
       # use cat to read /sys/firmware/vpd/ro/region
      -allow locale shell_exec:file rx_file_perms; # 这个是 system 下面的 sh
      +allow locale vendor_shell_exec:file rx_file_perms; # 这个是 vendor 下面的 sh
       
       # execute toolbox/toybox
      -allow locale toolbox_exec:file rx_file_perms;
      +allow locale vendor_toolbox_exec:file rx_file_perms;
      ```

   3. 继承了不兼容的属性

      我还不知道谷歌为什么会写成这样…… 一次给这么多权限真的没问题？不过反正 permissive 了不管了。

      ```diff
       # init_regions.sh reads region from vpd and sets
       # ro.product.locale property
      -type locale, domain, device_domain_deprecated; # device_domain_deprecated 里的 allow 贼夸张
      +type locale, domain;
      ```

   4. 天意不可违

      NVIDIA 平台难得出现个预编译的 blob，是管安全的（tlk）。它会在 /data/misc 下创建一些文件，而 Treble 要求在 /data/vendor 下，这就属于天意不可违了，只能直接把对应条目注释掉然后改 permissive.

      ```diff
      -allow tee nv_tee_data_file:dir create_dir_perms;
      -allow tee nv_tee_data_file:file create_file_perms;
      +#allow tee nv_tee_data_file:dir create_dir_perms;
      +#allow tee nv_tee_data_file:file create_file_perms;
      ```

   5. crash_collector 模块专属

      这个模块因为要从内核启动收集数据（在 `kernel.core_pattern` 后面加 `| /system/bin/crash_dispatcher64 xxx`）所以 transition 比较复杂，我是没找到合适的方案，所有相关的规则全注释了。此外，由于个人不太喜欢这个调用方式，所以这个模块也没移进 vendor.

#### log.h 的警告

会有提示说 cutils/log.h 已经 deprecated，建议使用 android/log.h 或者 log/log.h. 前者是后者的一个（我发现几乎没人用的）子集，所以在 device tree 里 sed 一下把 cutils/log.h 都替换成 log/log.h 就好了。

### 测试

废话少说，肯定炸了。

#### 图形驱动不能加载

有一些库，例如 libdrm.so，是 vendor_available: true 的，但如果没有 vendor module 链接它，它就不会被编译到 vendor 里面去，导致 NVIDIA 驱动里的一些库无法被 dlopen. 所以我做了一个没有代码的 fake vendor module 来链接它，保证 libdrm.so 能装进 vendor. 这操作跟 8.0 升 8.1 时 libhidltransport 消失的解决方法非常像。再次证明了“Google 认为只要不改代码重新编译就能解决的问题都不叫违反 Treble 的问题”。

但是，很遗憾，除了上面这个很容易解决的库以外，还有大量模块（我记得一个 libnvwsi.so，awsl）会链接不在 VNDK 里面的库，例如 libgui.so. logcat 里面一刷一大片 dlopen failed.

除了 logcat 里显而易见的问题，还有一种是给个 EGL_BAD_ALLOC 或者 EGL_NOT_INTIALIZED 就没了的，导致 surfaceflinger 和 hwcomposer 接龙爆炸。这种要 strace surfaceflinger 去看，会发现它在 vendor 下面找一个 so 没找到就崩了。补了一万个库到 vendor 下面以后，你会发现最后还是来到了 libgui.so 面前 :-)

尽管我上面为了开 Treble 做了这么多努力，对8起用不上了。老实把 `PRODUCT_FULL_TREBLE_OVERRIDE` 和 `BOARD_VNDK_VERSION` 去掉，然后按 Google 给的 7.0- 升 9.0 的建议加上 `PRODUCT_TREBLE_LINKER_NAMESPACES_OVERRIDE := false` 吧。

**注意**：我看到其他的维护者有个骚操作，直接把不合要求的链接给去掉了…… 然而我这还有直接通过 dlopen 加载的，怕是骚不起来了。

到这里应该能开机了，不过一点也不 Treble……

### 编译 SAR 开启的镜像

我在官方上看了一圈 SAR 文档，寻思好像也没说 SAR 要求 `BOARD_VNDK_CURRENT := true` 啊，也没要求 Treble 必须得开。虽然 SAR 不上 Treble 心里感觉很违和，但不如先编译个试试？

虽然上面 Treble 失败了，但之前改的那些 SEPolicy 和头文件还是很能提升幸福感的。

这步在 device repo 里面要添加下面的修改：

```
BOARD_BUILD_SYSTEM_ROOT_IMAGE := true
```

再把**所有**的 fstab 里 system 的挂载点改到 /，不要像我一样改了系统的没改 receovery 的，都快编译完了报个 AssertionError.

听起来很简单？那么再看看内核里的更改：

```
51414806e615 (HEAD) initramfs: Add skip_initramfs command line option
c7fe50983017 UPSTREAM: lib/string.c: introduce strreplace()
9bf57a5a8c8d dragon: enable DM_ANDROID_VERITY for System-as-Root
f9d7c0fd90c6 dm: fix dm_substitute_devices()
fa7040deeb2e ANDROID: dm: Rebase on top of 4.1
1ca016a0186e dm thin: fix a race in thin_dtr
9112dc60e755 dm thin: fix missing out-of-data-space to write mode transition if blocks are released
8c190ee800a0 dm thin: fix inability to discard blocks when in out-of-data-space mode
b067640fe436 dm space map metadata: fix sm_bootstrap_get_nr_blocks()
373e32b2ec02 dm cache: fix spurious cell_defer when dealing with partial block at end of device
bf059b40fd64 dm cache: dirty flag was mistakenly being cleared when promoting via overwrite
43df343a1cd6 dm cache: only use overwrite optimisation for promotion when in writeback mode
826a9ccb3f82 dm crypt: use memzero_explicit for on-stack buffer
b2998cc20f29 dm bufio: fix memleak when using a dm_buffer's inline bio
606b089805e0 ANDROID: dm: android-verity: allow disable dm-verity for Treble VTS
b608141ff904 ANDROID: dm verity: add minimum prefetch size
efc9c9adb8fa ANDROID: dm: android-verity: Remove fec_header location constraint
fcd5bd9e087e ANDROID: dm: Fix symbol exports for dm target callbacks
29d31b120db1 ANDROID: dm: android-verity: Allow android-verity to be compiled as an independent module
2c75f96b5c59 ANDROID: dm: android-verity: Verify header before fetching table
bf737870f9a1 ANDROID: dm: allow adb disable-verity only in userdebug
66b5a8e65224 ANDROID: dm: mount as linear target if eng build
4ae0a79d7b36 ANDROID: dm verity fec: initialize recursion level
41c23e3afd6d ANDROID: dm verity fec: fix RS block calculation
f698477dad47 ANDROID: dm verity fec: pack the fec_header structure
4b905e5bdac3 ANDROID: dm verity fec: add missing release from fec_ktype
111439404d65 ANDROID: dm verity fec: limit error correction recursion
00d66bf44928 ANDROID: dm: use default verity public key
850735087357 ANDROID: dm: fix signature verification flag
c7220b9b5531 ANDROID: dm: use name_to_dev_t
2066060b6d8a ANDROID: dm: rename dm-linear methods for dm-android-verity
eecfdd39eff9 ANDROID: dm verity fec: add sysfs attribute fec/corrected
f0bd9b123aeb ANDROID: dm: Mounting root as linear device when verity disabled
fadc80622a4d ANDROID: dm-crypt: Remove WQ_NON_REENTRANT flag.
e0628e390195 ANDROID: dm-android-verity: Rebase on top of 4.1
df3c9324201a ANDROID: dm: Add android verity target
56eef7114261 CHROMIUM: dm: boot time specification of dm=
73306726647d ANDROID: dm-crypt: run in a WQ_HIGHPRI workqueue
d036f81182ff ANDROID: dm-verity: run in a WQ_HIGHPRI workqueue
3e9fe6574096 UPSTREAM: dm crypt: sort writes
e5354eba2258 UPSTREAM: dm crypt: offload writes to thread
b0f55490c6a4 UPSTREAM: dm crypt: remove unused io_pool and _crypt_io_pool
890dfe6c41db UPSTREAM: dm crypt: avoid deadlock in mempools
4fab7f49420e UPSTREAM: dm crypt: don't allocate pages for a partial request
a533dc3b1d35 UPSTREAM: dm verity: add ignore_zero_blocks feature
1d505e99d24b UPSTREAM: dm verity: add support for forward error correction
dcdfc5499ef7 UPSTREAM: dm verity: factor out verity_for_bv_block()
42b2efaa1505 UPSTREAM: dm verity: factor out structures and functions useful to separate object
2da7522a10de UPSTREAM: dm verity: move dm-verity.c to dm-verity-target.c
cf6cfbcd5699 UPSTREAM: dm verity: separate function for parsing opt args
dd5ff960c546 UPSTREAM: dm verity: clean up duplicate hashing code
019b4ff09944 ANDROID: dm verity: port upstream changes to 3.18
6bd75c8a7a64 dm-verity: Add modes and emit uevent on corrupted blocks
acc78194a612 Revert "CHROMIUM: dm: Add support for devices by PARTUUID/PARTNROFF."
52d296594850 Revert "CHROMIUM: md: dm-verity fixes chromeos needs"
40a235db10d3 Revert "CHROMIUM: dm: boot time specification of dm="
eaab5774fed0 Revert "dm_verity: remove pre-.33-skip_spaces"
83a3f5dfb5cd Revert "CHROMIUM: make dm= boot path wait for devices"
c37b5e727ec5 Revert "CHROMIUM: verity: build fix for init/do_mounts_dm.c"
dc017bb1f37d Revert "CHROMIUM: init: don't dm_substitute_devices()."
96a8a79d4027 Revert "Enhanced do_mounts_dm to handle multiple devices"
d592110fb0c3 Revert "CHROMIUM: dm: fix newlines in printouts"
044d2070b131 Revert "CHROMIUM: dm: pass up rotational flag"
eefab997b149 Revert "CHROMIUM: dm: bootcache Device mapper for bootcache"
7ba7cee51d98 Revert "CHROMIUM: dm: bootcache Device mapper for bootcache"
c646ba02b1fd Revert "CHROMIUM: dm: bootcache communicate alignment"
b9d132b90c1a Revert "CHROMIUM: md: bootcache check metadata and data lengths"
eedd34876341 Revert "CHROMIUM: md: bootcache don't delete sys files"
5ea638ad8aa5 Revert "CHROMIUM: md: bootcache if init fails, mark invalid"
3ad12202f0c1 Revert "CHROMIUM: md: bootcache save string args"
6beeaec4930b Revert "CHROMIUM: md: fix dm-bootcache string compare length"
4fea77b63d05 Revert "CHROMIUM: md: bootcache compile fixes for 3.8"
e59d3a4cff07 Revert "CHROMIUM: md: Fix up verity and bootcache uuid matching for kernel 3.8"
b2200af3d750 Revert "CHROMIUM: verity: fix UUID checking for VMTests"
268d97c8e65d Revert "CHROMIUM: verity/bootcache: fix warning on bootcache"
c0f2ba9bb5dd Revert "CHROMIUM: verity: bring over dm-verity-chromeos.c"
377c207222f6 Revert "CHROMIUM: md: dm-verity: call to verity_error missing"
2a7da6ae33fb Revert "CHROMIUM: md: dm-verity: error handling for bad block"
877d234913b9 Revert "CHROMIUM: md: dm-verity fixed setting error_behavior"
ef5389bc4bbd Revert "CHROMIUM: md: dm-bootcache: reinitialize bio structure"
efe912308cc5 Revert "CHROMIUM: md: dm-bootcache, dm-verity, dm-verity-chromeps - support for PARTUUID"
0b286fe320b2 Revert "CHROMIUM: md: dm-bootcache: validate number of trace records"
51a022dc9c05 Revert "CHROMIUM: md: dm-bootcache: restricted trace records to a fraction of total memory"
67aa7e0dc4f8 Revert "CHROMIUM: drivers: md: changed device lookup to use linux routine."
7292205e2576 Revert "CHROMIUM: init: made devt_from_partuuid an extern"
6e1b07748265 Revert "FROMLIST: init: Export name_to_dev_t and mark it const"
203b87751735 Revert "CHROMIUM: drivers: md: removed uuid code"
29089933f30f Revert "dm-verity.c: restore transient error handling"
d9a53f309bd9 Revert "CHROMIUM: Fixup verity_chromeos and bootcache for 3.14"
a257a6ce539b Revert "CHROMIUM: md: dm-bootcache: use utsname()->version as timestamp"
768060cbc89f Revert "verity: Add error path for ubiblock-backed dm-verity"
77bd85958b91 Revert "CHROMIUM: dm_verity: speed up boot when not using verity"
96a9e4c35527 Revert "FROMLIST: dm: Get devices using name_to_dev_t"
a13047bbb4db Revert "CHROMIUM: md: verity-chromeos: get_boot_dev by UUID before rootdev"
7a7f61a77d2b Revert "CHROMIUM: md: verity: Make DM_VERITY_CHROMEOS tristate"
6d6f230c6f16 Revert "dm ioctl: prevent stack leak in dm ioctl call"
5a6bec7e7ee7 (mkp) arm64: DT: disable early system partition mounting
80a59ef9c289 arm64: DT: System-as-Root bootargs
```

哈哈哈哈哈哈哈哈哈哈哈刺激吧

Pixel C 最开始是个 ChromeOS 设备，所以有很多关于 ChromeOS 的 verity 的修改。我这里为了适应 SAR 强制加 system verity 的需求，只能把 ChromeOS 的修改全部 revert 掉，再把 Google Source 上 kernel/common 3.18 分支 drivers/md 的全部有关更改再 pick 过来。

> 怎么确定哪些更改是 ChromeOS 引入的、哪些是升内核版本合的、哪些是 Android 引入的？
>
> oh-my-zsh 的 glog 简直好用到爆，可以清晰地看到哪些提交是作为一整个分支合进来的。滚一下就找到 ChromeOS / Android 最初的那一串提交了，一个复制粘贴就可以解决 revert / pick 的问题。

至于最前面的 arm64: DT: System-as-Root bootargs 和 arm64: DT: disable early system partition mounting，则只是添加 Google 要求的 `dm="..." skip_initramfs init=/init` 和 disable 掉 DTS fstab 的 system 而已。前者是为了让内核不要使用内核自带的 initrd 而是把 system 分区初始化成 rootfs，后者是避免重复挂载导致 init 在 early stage 失败重启。

感谢 Google 还在维护 3.18 的 common kernel.

此外，还有些小细节更改：

（来自 dianlujitao，我针对 Pixel C 做了修改，因为它的 bootloader 毛参数都不会传）

```diff
diff --git a/drivers/md/dm-android-verity.c b/drivers/md/dm-android-verity.c
index 3d9e69b388a4..bbe18121e1b2 100644
--- a/drivers/md/dm-android-verity.c
+++ b/drivers/md/dm-android-verity.c
@@ -104,23 +104,17 @@ static inline bool default_verity_key_id(void)
 
 static inline bool is_eng(void)
 {
-       static const char typeeng[]  = "eng";
-
-       return !strncmp(buildvariant, typeeng, sizeof(typeeng));
+       return true;
 }
 
 static inline bool is_userdebug(void)
 {
-       static const char typeuserdebug[]  = "userdebug";
-
-       return !strncmp(buildvariant, typeuserdebug, sizeof(typeuserdebug));
+       return true;
 }
 
 static inline bool is_unlocked(void)
 {
-       static const char unlocked[] = "orange";
-
-       return !strncmp(verifiedbootstate, unlocked, sizeof(unlocked));
+       return true;
 }
 
 static int table_extract_mpi_array(struct public_key_signature *pks,

```

这是为了跳过验证直接挂 system，不然还要搞 verity 的事情太烦了。

### 测试

尽管上面 Treble 的操作已经让系统规范了很多，但还是起不来。

#### nouveau panic

看 /sys/fs/pstore/console-ramoops（说起这个文件真是一把辛酸泪）可以发现是 nouveau 里面 panic 了，再顶着非常朋克的乱码往前看可以发现 N 卡 firmware 没加载。以前这些 firmware 是放在 initrd 里面的，内核一启动就能用。现在 system 分区替代了 initrd，即使它的挂载提到比 init 还早了，也毕竟比不上 initrd，nouveau 需要的时候 system 还没挂，所以 N 卡就挂了。为了追速度，我直接把原先放在 initrd 里面的 firmware 编译进内核了，可以说是比 initrd 更早就可用，问题解决。这才让内核增大了 200KB 左右，很方便，但是这样就造成许可证不统一，无法公开代码了……

到这里就有了一份开启 SAR 的能用的 Android P 镜像。这证明不开 Treble 也可以 SAR，不过是不是有点**“形式主义”**呢？

## 刷 Pixel 2 的 system 镜像

刺激的部分开始了。至于为什么不是 Pixel 2 XL 等机型？因为一开始在 OnePlus 5 上 Pixel 2 XL 失败了，留下了巨大的心里阴影，经电大建议换了 Pixel 2. 虽然 OnePlus 5 上仍然没成功，但至少没那么蛋疼了。

好了，直接开炸吧。

#### 未知原因的秒重启

一进 Google 画面就重启，并且没有 /sys/fs/pstore/console-ramoops 了，这可真是让人头大。以前遇到这种情况只能反复重启设备，期待有那么一次内存没被损坏，能留下这个文件。好在重启了几次看到了一份，说是 /vendor 下面 sepolicy 的配置文件找不到。

我寻思那是开了 Treble 才会分离的 sepolicy，不开 Treble 怎么会有呢？又想了一下，之前在 Google 上看 VNDK 的时候，好像看到一个不开 Treble 也可以强制分离 sepolicy 的选项。搜了一下确认是有，`PRODUCT_SEPOLICY_SPLIT_OVERRIDE := true`. 这样确实可以产生完整的 /vendor/etc/selinux，去 recovery 下 push 进去就可以解决了。

#### 未知原因的卡死

卡死在 Google 画面，只能电源 + 音量下重启，并且没有 /sys/fs/pstore/console-ramoops. 更让人头大的是这次怎么重启都只有完全不可读的 pmsg 了，console 始终出不来。加了 loggy 也没用，不过加 loggy 时发现一个问题：/cache 是个 -> /data/cache 的链接。cache 分区本来也没挂在 /data 下，所以我把这个链接删了，重建了个 /cache 目录。

即便是 cache 理论上可用了，但 loggy 仍然不出数据。在 OnePlus 5 上我已经在 init.rc 里到处点灯来确定进度了，但是 Pixel C 没灯可点啊，于是选择用 `mkdir /cache/xxx` 和 `exec - /system/bin/sleep 20` 来确定进度。真实落魄。

> 即便是到了这种看起来走投无路的地步，也还是有坑：sleep 必须写完整路径，否则无法执行。可能在 init 的文档里有写吧。
>
> 另一个类似的问题是，vendor 里的 loggy 脚本也必须执行 /system/bin/logcat，直接使用 logcat 也是无法执行的。

无心插柳柳成荫，sleep 20 后竟然每次重启都能看到 console-ramoops 了。我已经不知道该是什么表情了，更不知道为什么 sleep 能修掉 pstore 的问题，赶紧看内容吧。

可以看到是 Keymaster HAL 疯狂挂掉，然后 init 疯狂重启它。console 上看不到原因，只能先把它从 vendor manifest 里注释掉了。在这之后 loggy 就能往 cache 写日志了。可能这个问题卡着 fstab mount 了吧。

看打出来的日志，问题是缺 libpuresoftkeymaster.so，从 P 里 32 位 64 位全推一遍，解决。

#### /data read-only

现在能看到 logcat 了，方便多了。只是每次重启去 recovery 看日志要 30s 比较不爽。

看到 /data 是 read-only 的，基本不能往里写数据，启动过程走不下去。一看 data 虽然格式化了，但 fstab 里还是 FDE 的。FDE 在 P 里就得额外提交才支持，Q 相比更是这样。把 fstab 里的加密参数去掉，解决。相比前两个真是太轻松了。

#### 缺库

跟着 logcat 的 dlopen failed 把各种 HAL 的库补全即可。binary 的数量很多，要补的 library 也不少，其中很多是旧版本的 HIDL native interface…… 说真的，Google 真不觉得“重新编译就能用”是一个很可笑的说法吗？

并且每补一个库都要重启等 30s 看效果，然后再等 30s 进 recovery 看新的日志又缺了哪些库。敲键盘只有两行文字，实际上又不知道花了多久。（其实我有一个递归找出所有依赖的脚本，但是最近太懒了没用，实际上如果用的话应该能节省大量时间……）

#### 还是缺库

这跟上面的缺库有什么不同呢？因为 Q system 开了 VNDK，所以 vendor 用的动态链接库有限制，即使 push 到 system 也没用。前面已经说了这是不行的。需要把 VNDK 干掉。

我这里走点偏门：用 P 的 /system/etc/ld.config.txt 放进 Q 里，再把 ld.config.Q.txt 的内容改成跟它一样。不想管到底哪个起作用，都放进去就对了。我的 30s 已经浪费到想吐了。

#### 反正就是不走了

已经看不到新的缺库错误了，但启动过程就是卡着不往下走。看 ps，有两个进程应该退出的没退出（iptables 系列），想杀了它们，但是 Q system 的 adbd 不是 userdebug 版无法 root 杀不掉。重启替换它们为 P 版，不行。删掉，不行。爆粗，可以。

拿出已经刷回 P 的 OnePlus 5 看启动日志，对比了一下，最后一个 trigger on nonencrypted 里面两条指令都成功执行完了啊没有什么不一样啊，为什么 Java 层的毛都没起来？搜了一下 zygote，我（文明用语）为什么 Q 上 ro.zygote 没有定义？zygote 整个服务都没有了好吗？你知道我看到 `service 'zygote' not found` 的时候有多炸毛吗？

算了，不要无能狂怒了。我也不知道为什么 ro.zygote 没定义，手动把 init.rc 里 import 的地方改成 zygote64_32，问题解决，可以继续走了。

#### Apex Legends

之前在微博上见过有大佬说 Q 里面有个新的叫 APEX 的玩意，好像跟 Magisk 的模块有一丢丢像，都是 loop device 的形式挂载的，但 apex 是 system 里面有链接指过去，而不是自己覆盖到 system 之上。据说这是为了热更新核心组件，我看 libc libm 之类的都在里面，确实挺适合模块化更新。这个方法在 Android 原生系统里面看起来真的挺 legendary 的。

但是，它现在在给我搞事。libnativeloader 报错说 `Error finding namespace of apex: no namespace called runtime` . 查了文档，说是每个 apex 有自己的 namespace，具体 namespace 名字是从 apex 报名截出来的，例如 com.android.runtime.release.apex 的 namespace 是 runtime、com.android.media.apex 是 media 等. 看这是 libnativeloader 里面的提示，应该是 linker namespace 吧。找来 Q 原本的 ld.config.Q.txt 一看，确实里面有配置这些 namespace，通通拷过来，解决。

> 我在做 OnePlus 5 时就是到这一步卡死了导致放弃，当时以为是 Linux kernel 那边的什么 namespace，根本不知道怎么跟进。这次算是心态好了一点，搜了日志，才解决掉。
>
> 并且一开始做的时候手残其中一个 ld.config.txt（或者是 Q 的，忘了）只改了一半，导致又浪费了两三个 30s.

#### 设置向导：汝可识得此阵？

这次终于看到一个可操作的界面，十分高兴。顶着蓝牙循环崩溃去走设置向导，注意到 logcat 里面一秒一万条的 Java GC，但好像不影响用，我先进主屏看看再说。

可是设置向导完了以后又软重启回设置向导了。看日志说是 Java heap 不足。这我就有经验了，一般是 dalvik.vm.* 设置不对，内存吃没了。虽然 Pixel 也只有 4G，但多 1G 也是 1G 吧。

重启去 recovery 发现这不是一般情况：dalvik.vm.* 的设置根本就没有，而不是不对。这就比较厉害了，我上次遇到还是因为 device tree include 错了导致的，不知道谷歌在 Q 上把它弄哪去了。可能是在 vendor 里面吧。

> 写到这里，想起上面 ro.zygote 没了，觉得 ro.zygote 跟 dalvik.vm.* 可能是在同个地方，然后这个文件整个丢了吧。

把 P 里面的这些属性带 hwui 一起复制到 Q 上，谢天谢地终于过了，进 launcher 了。

#### Wi-Fi

一开始是连不上的，logcat 提示 Supplicant HIDL 不支持 KeyMgmt 啥的方法，跟 OnePlus 5 刷其他人的 GSI 现象一样。我估计是 wpa_supplicant 加了新接口，直接从 Q vendor 里撸了一个过来替换掉，再在 manifest 里按 Q 的升级 Supplicant HIDL 的版本，解决了。

#### 蓝牙

根据电大的说法，蓝牙部分是缺个新的 HIDL 所以空指针崩溃。不管能不能修，我暂时是懒得修了……

#### 相机

一开始它是因为链接 libgui 被我给注释掉了，现在就可能能用了。有时间加回来看看吧。

#### 音频 & 编解码

我知道它炸了，但没看到日志。有缘再修。

## 后记

刷 Pixel C 之前也做了 OnePlus 5 的，但是没有成功，文中也多次提到了，主要是链接库太烦了。后来想着 NVIDIA 的 blob 不多，依赖应该比较好处理，所以又尝试了 Pixel C 的，终于算是成功一次，感觉自己不那么垃圾。

其中还是挺有些运气成分吧，比如我都忘了 Q 上 hwcomposer.drm.so 是啥时候放进去的了（它在 system 里的依赖很多，又是 external/ 的，没法移进 vendor），上文都没有写，结果它居然一直都没给我找事。搁其他平台，图形部分往往不会走得这么顺利。当然也可能是因为 nouveau + hwcomposer_drm 这样的开源组合本来就没什么事……

最近心态确实比较爆炸，公司的项目做太久很疲乏，好像也不是我一个人这样；第一次搞 Q 也失败了；花不少钱买的俩手机一个预售一个还在海淘路上晃，货还差几天到手又看到近期有屏幕坏点、指纹失灵之类的问题，虽然数量很少但心里也很慌；现在的手机难得是个 4K LCD 屏，结果显示灰色时屏幕中间有条细线的问题又复发了…… 还有一堆事情悬而未决心里总是吊着些东西，一上网看到各路新闻也很影响心情，Pixel C 这真是难得一件开心的事了。生活不易啊。