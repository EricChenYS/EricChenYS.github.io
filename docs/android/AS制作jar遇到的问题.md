---
layout: default
title: AS制作jar遇到的问题
description: AS制作jar时遇到implementation引入的其他jar无法被打包进入jar
---
# 问题描述
使用Android Studio的gradle编译jar（apply plugin: 'java-library'）时，发现不管是implementation还是compileOnly引入的jar里面的代码都不会被打包引入jar
期望结果：implementation引入的会被打包进来，compileOnly引入的不被打包进来

# 原因
## 1. Gradle依赖配置特性
implementation配置的依赖是编译+运行时依赖，但默认不会被打包到输出的JAR中
Java Library插件设计的标准JAR包仅包含项目自身编译的类文件
这是Gradle的标准行为，遵循Java库的常规打包规范
## 2. JAR任务默认行为
默认jar任务只打包src/main/java编译输出的.class文件
不包含任何依赖项（包括本地JAR和远程库）


# 解决方案
集成gradle的插件Shadow可以创建FAT JAR
https://gradleup.com/shadow/

## project build.gradle
```java
    repositories
        // 增加
        maven {
            url = uri("https://plugins.gradle.org/m2/")
        }
        mavenCentral()
    }

    dependencies {
        classpath "com.android.tools.build:gradle:7.4.2"
        classpath 'com.gradleup.shadow:shadow-gradle-plugin:8.3.4'  //增加
    }
```

## gradle wrapper
distributionUrl=https\://services.gradle.org/distributions/gradle-8.5-bin.zip

## jar的build.gradle
```java
apply plugin: 'java-library'
apply plugin: 'com.gradleup.shadow' // 增加
```
## 编译方法
jar的Gradle的Tasks里面会出现shadow，点击shaowJar就会编译出jar
