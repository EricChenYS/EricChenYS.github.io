---
layout: default
title: Android DIY ROM
description: 如何制作Android官改ROM
---

# 一, 步骤
## 1). 从手机里面获取img
1. adb shell
2. cd dev/block/platform/bootdevice/by-name （不同Android版本，路径可能存在差异）
3. ls -l
   lrwxrwxrwx 1 root root   21 2010-01-01 08:00 super -> /dev/block/mmcblk0p44
4. dd if=/dev/block/mmcblk0p44 of=/data/local/tmp/super.img
5. adb pull data/local/tmp/super.img .

 
## 2). 解包
imjtool.ELF64 super.img extract

记录group name，后面需要用到

注：从out下面拷贝的super.img可能需要解包2次（即对解包的包再次执行上面操作）


## 3). 改img（以改system_a.img为例，改其他img操作相同）
### ext4文件系统
1. mkdir system
2. sudo mount -t ext4 -o loop system_a.img system
3. 进入到挂载目录删除（或修改）文件
4. sudo umount system

注：也可以在root的手机里面挂载（例如：mount -t ext4 -o loop sdcard/system_a.img data/local/tmp/system ）

### erofs文件系统
1. erofs解包
sudo fsck.erofs --extract=./extracted --no-preserve --force --overwrite product_a.img
2. 删除目标文件
3. erofs重打包
sudo mkfs.erofs -zlz4  -U00000000-0000-0000-0000-000000000000 --ignore-mtime lz4.product.img ./extracted/


## 4). 获取各个img的大小（下面方法选一）
stat -c '%n %s' *.img 

ls -l *.img


## 5). 重打包
sprd:

lpmake --metadata-size 65536\
 --device-size=5872025600\  根据xml中的super分区size调整
 --metadata-slots=3\
 --group=group_unisoc_a:5177221120\
 --partition=system_a:readonly:1643384832:group_unisoc_a\
 --partition=system_ext_a:readonly:963981312:group_unisoc_a\
 --partition=vendor_a:readonly:861888512:group_unisoc_a\
 --partition=product_a:readonly:1688854528:group_unisoc_a\
 --partition=vendor_dlkm_a:readonly:19111936:group_unisoc_a\
 --image=system_a=./system_a.img\
 --image=system_ext_a=./system_ext_a.img\
 --image=vendor_a=./vendor_a.img\
 --image=product_a=./lz4.product.img\
 --image=vendor_dlkm_a=./vendor_dlkm_a.img\
 --group=group_unisoc_b:0\  可以去掉
 --partition=system_b:readonly:0:group_unisoc_b\
 --partition=system_ext_b:readonly:0:group_unisoc_b\
 --partition=vendor_b:readonly:0:group_unisoc_b\
 --partition=product_b:readonly:0:group_unisoc_b\
 --partition=vendor_dlkm_b:readonly:0:group_unisoc_b\
 --image=system_b=./system_b.img\
 --image=system_ext_b=./system_ext_b.img\
 --image=vendor_b=./vendor_b.img\
 --image=product_b=./product_b.img\
 --image=vendor_dlkm_b=./vendor_dlkm_b.img\
 --sparse \
 --output ./super.new.img


 若出现[liblp]Partition should only have linear extents: system_b，可把group_unisoc_b全去掉

Mtk:

lpmake --metadata-size 65536\
 --device-size=8589934592\
 --metadata-slots=3\
 --group=main_a:6872780800\
 --partition=product_a:none:3571236864:main_a\
 --partition=system_a:none:2819018752:main_a\
 --partition=vendor_a:none:482525184:main_a\
 --image=product_a=./product_a.img\
 --image=system_a=./system_a.img\
 --image=vendor_a=./vendor_a.img\
 --group=main_b:220397568\
 --partition=product_b:none:0:main_b\
 --partition=system_b:none:220397568:main_b\
 --partition=vendor_b:none:0:main_b\
 --image=product_b=./product_b.img\
 --image=system_b=./system_b.img\
 --image=vendor_b=./vendor_b.img\
 --sparse \
 --output ./super.new.img

注：group main_a和main_b在解压的时候可以看到product_a，system_a，vendor_a等的大小由步骤4计算出来group后面size是该group下img size的总和
[图片]


./lpmake --metadata-size 65536\
 --device-size=11781799936\
 --metadata-slots=3\
 --group=main_a:7551188992\
 --partition=product_a:none:3211993088:main_a\
 --partition=system_a:none:865472512:main_a\
 --partition=system_ext_a:none:2102554624:main_a\
 --partition=vendor_a:none:1371168768:main_a\
 --image=product_a=./product_a.img\
 --image=system_a=./system_a.img\
 --image=system_ext_a=./system_ext_a.img\
 --image=vendor_a=./vendor_a.img\
 --sparse \
 --output ./super.new.img

简单步骤
1. 先从手机里面拿到super.img
cd dev/block/platform/bootdevice/by-name 
lrwxrwxrwx 1 root root   21 2010-01-01 08:00 super -> /dev/block/mmcblk0p44
dd if=/dev/block/mmcblk0p44 of=/data/local/tmp/super.img
2. 然后mout，修改内容，unmout
3. 然后刷这个susper.img


# 二, 工具
## 1. imjtool
http://newandroidbook.com/tools/imjtool.html

从手机里面获取img的方法如下，使用下面方法也可以获取到其他img，例如：susper.img
[图片]

注：不同android版本路径可能有差异
dev/block/platform/bootdevice/by-name

## 2. lpunpack_and_lpmake
https://github.com/Exynos-nibba/lpunpack-lpmake-mirror/tree/Linux-debian/binary

## 3. Busybox
https://github.com/xerta555/Busybox-Binaries

## 4. erofs-utils
https://github.com/sekaiacg/erofs-utils/releases/tag/v1.6-230720


# 三, 参考文档
https://blog.senyuuri.info/posts/2022-04-27-patching-android-super-images/

https://forum.xda-developers.com/t/guide-universal-guide-for-making-your-partitions-inside-super-read-writable-again.4483933/

erofs格式img
https://github.com/sekaiacg/erofs-utils 已编译

https://git.kernel.org/pub/scm/linux/kernel/git/xiang/erofs-utils.git/snapshot/erofs-utils-1.6.tar.gz
