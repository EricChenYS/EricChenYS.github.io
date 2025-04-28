---
layout: default
title: aidl生成jar
description: aidl如何生成jar
---

### 步骤：
## 1. aidl生成java
# 方法一
放到Android Studio生成jar，然后编译，会在build/generated/aidl_source_output_dir/下面生产java文件
可能存在的问题：生产的JAR里面可能不包含DESCRIPTION成员（可能有gradle版本有关）。可以采用方法二生成DESCRIPTION成员
# 方法二
通过android sdk里面的aidl命令
aidl xx.aidl
如果aidl文件依赖其他的aidl文件，需要使用"-I"参数
[图片]
参考https://blog.csdn.net/yinminsumeng/article/details/129324917

## 2. java编译生成class
可以放到Android Stduio里面编译，会在build/intermediates/javac下面生成class文件

## 3. class打包成jar
jar -cvf xxx.jar dir 
