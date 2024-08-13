---
title: InteliJIDEA 配置 Minecraft 插件开发实时调试环境, 并进行断点调试
date: 2024-08-12 10:05:12
tags: Minecraft, Java
---

### 前言

我在开发 MC 服务器插件的时候, 常常因为需要在开发界面和命令行窗口之间来回切换而感到烦恼, 截止到本篇文章成文之前, 我认识的插件开发者基本都是先把插件构建好, 然后将构建产物**手动复制到服务端插件目录**进行调试, 这样的效率实在是过于低下. 于是我就思考有没有一种方式能够只在 IDEA 内就完成下面这些步骤: 

1. 插件的构建
2. 复制构建产物到服务端插件目录中
3. 运行调试服务器
4. 进行实时的断点调试

于是我就开启了漫长的网络搜索过程, 终于找到了一篇[比较不错的文章](https://blog.csdn.net/qq_41042178/article/details/123175466)满足了我的需求, 因此本文的内容只是基于这篇文章的学习记录

### 开发环境的搭建

如果已经配置好了开发环境, 可以跳过这段内容.
开发环境采用 IDEA 捆绑的 Gradle 环境, Java 为 JDK17, 使用的服务端 API 为 SpigotAPI, 基本代码如下:

```groovy

//...

repositories {
    maven {
        name = "spigotmc-repo"
        url = "https://hub.spigotmc.org/nexus/content/repositories/snapshots/"
    }
}

dependencies {
    compileOnly "org.spigotmc:spigot-api:1.20.4-R0.1-SNAPSHOT"
}

//...

```

### 插件的构建

如果你的插件没有依赖任何外部库, 直接使用 Gradle 项目自带的任务 jar 即可完成构建

![Jar 任务位置](https://cdn.jsdelivr.net/gh/ed333a/blogcomments/images/blog/20240812020512/1.png)

如果你的插件依赖了外部库, 则需要使用 `shadowJar` 的 Gradle 插件来打包外部库, 在 `build.gradle` 文件中添加以下代码:

```groovy
//...
plugins {
    //...
    id "com.github.johnrengelman.shadow" version "7.1.2"
    //...
}
//...
```

然后重载一下 Gradle 项目, 这时候就会出现 `shadowJar` 任务了

![ShadowJar 任务](https://cdn.jsdelivr.net/gh/ed333a/blogcomments/images/blog/20240812020512/2.png)

需要注意的是, 不对依赖项进行包位置重定向的话, 直接使用 shadowJar 任务, 会直接把依赖库打包到与插件包名同级的位置, 以下是我的插件对依赖库进行重定向之后的效果

![包位置重定向](https://cdn.jsdelivr.net/gh/ed333a/blogcomments/images/blog/20240812020512/3.png)

进行包位置重定向也很简单, 只需要在 `build.gradle` 中配置如下代码, 第一个参数是你需要重定向的包, 第二个参数指向你重定向的位置

```groovy
shadowJar {
    relocate("org.a.b.c", "to.your.destination");
}
```

除此之外, `shadowJar` 的 `minimize()` 方法还可以将依赖库中没有用到的类进行去除, 缩小构建产物体积. 最终构建好的产物会位于 `build/libs` 文件夹内

### 配置调试用的 MC 服务器

为了方便(主要还是我太懒了, 不想再重新配置一个服务端了), 我这边直接换到我之前的一个项目了, 服务端的位置放置没啥要求, 你自己记得就行, 我这边为了方便就直接放到项目目录内了. 服务端内的文件没什么特别的东西, 就是正常的服务端就行.

![服务端内文件](https://cdn.jsdelivr.net/gh/ed333a/blogcomments/images/blog/20240812020512/4.png)

### 为 gradle 项目添加一个复制任务

服务端配置好了以后, 我们还需要在 gradle 项目中添加一个复制构建产物到服务端目录的任务, 代码如下 

```groovy
tasks.register('copy', Copy) {
    from "build/libs"  // 源文件夹, 在这里是构建产物的文件夹 
    into 'server/paper-1.20.4/plugins'         // 目标文件夹, 在这里是调试服务端的插件文件夹
    include "YourArtifactName-${version}.jar"  // 可选：只复制特定类型的文件,支持正则表达式 在这里只规定该名称的文件可以被复制到目标文件夹
}

```

配置好后重新加载 gradle 项目, 看看右侧任务列表中是否有 `copy` 任务, 存在即为配置成功.

![copy 任务位置](https://cdn.jsdelivr.net/gh/ed333a/blogcomments/images/blog/20240812020512/5.png)

### 在 IDEA 中配置调试任务

在 `Run >> Edit Configurations` 中:

![配置调试任务](https://cdn.jsdelivr.net/gh/ed333a/blogcomments/images/blog/20240812020512/6.png)

该界面往下拉会有一个 `Before Launch` 列表， 表示在执行该任务时会先执行列表中的任务, 执行顺序为自上而下， 将我们之前所提及到的构建插件、复制插件任务添加到这里, 之后直接执行该任务就可以实现插件的构建、复制、运行服务端进行调试了

![Before Launch](https://cdn.jsdelivr.net/gh/ed333a/blogcomments/images/blog/20240812020512/7.png)

### 运行、运行以及调试

![按钮介绍](https://cdn.jsdelivr.net/gh/ed333a/blogcomments/images/blog/20240812020512/8.png)

我们来在 `onEnable()` 方法内写一个测试用的程序, 然后点击 "运行以及调试" 按钮来进行断点调试

{% asset_img 9.png "断点示例程序" %}
![断点示例程序](https://cdn.jsdelivr.net/gh/ed333a/blogcomments/images/blog/20240812020512/9.png)

调试后可以看到我们的程序成功在断点位置停了下来

![断点测试结果](https://cdn.jsdelivr.net/gh/ed333a/blogcomments/images/blog/20240812020512/10.png)