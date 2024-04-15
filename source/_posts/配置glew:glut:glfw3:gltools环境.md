---
title: Mac下使用OpenGL——配置glew/glut/glfw3/gltools环境
date: 2016-09-02 13:05:00
tags: "OpenGL"
categories: "OpenGL"
---

glew/glut/glfw3/gltools它们都是OpenGL的扩展或工具，其中glut是mac自带的，这里就不用讲了，直接就可以用。

### 一、安装homebrew
brew 的官方网站： [http://brew.sh/]("http://brew.sh/")   
在官方网站对brew的用法进行了详细的描述，安装方法：  在Mac中打开Termal:  输入命令：
```
/usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
```
下面是homebrew的一些命令：
brew search  搜索软件包
brew install  安装软件包
brew uninstall   卸载软件包
brew info  查询软件包信息
brew list  查询已经安装的软件包
brew update  更新
brew deps  显示包依赖

### 二、利用homebrew安装cmake
输入：
```
brew install cmake
```

如果一切正常就到到下一步，这里可能报下面错误：

>Error: The brew link step did not complete successfully
The formula built, but is not symlinked into /usr/local
Could not symlink share/man/man7/cmake-buildsystem.7
/usr/local/share/man/man7 is not writable.

解决方法：
先执行：`sudo chown -R $(whoami) /usr/local`
再执行：`brew link cmake`

### 三、安装glew/glfw3
执行命令：
```
brew install glew
brew install glfw3
```
安装成功后，可以在`/usr/local/Cellar`目录下找到glew/glfw3的.a文件和头文件

### 四、下载编译gltools
下载链接：https://github.com/HazimGazov/GLTools
编译：
![](http://upload-images.jianshu.io/upload_images/1258499-744e792090eea60f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 五、Xcode使用我们安装好的gl扩展或工具
  * 第一种：直接在*/usr/local/Cellar*文件下找到glew/glfw3文件，在*/usr/local/include* 和*/usr/local/lib*文件下找到gltools，将头文件和库都拖进工程
  * 第二种：原文连接：https://zrz0f.com/2016/02/21/glfw/

### 六、装了gltools之后，使用上面的第二种，设置会简单很多
Xcode的Proferences > Locations > Source Trees 中

![](http://upload-images.jianshu.io/upload_images/1258499-a6643aae761fc196.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

里面的两个路径分别如下图：

![](http://upload-images.jianshu.io/upload_images/1258499-119e0398f46f3b05.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在Xcode项目中：

![](http://upload-images.jianshu.io/upload_images/1258499-75497a632e23e49e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

设置项目的Other Linker Flags：
![](http://upload-images.jianshu.io/upload_images/1258499-5c22ce81d6a693b6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

注意：如果你在项目中用到了gltools和glut，你还是要导入.a或framework文件，如下图：

![](http://upload-images.jianshu.io/upload_images/1258499-bd152f7f51f95c1d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

配置好了以后，关于OpenGL的glew/glut/glfw3/gltools就都可以用了

### 七、运行第一个OpenGL工程
创建一个Mac App，glfw的官网可以下载演示demo，下载glfw将文件中simple.c拖入工程中(如下图)，删掉main.m，然后运行，OpenGL的第一个工程就运行成功了！

![](http://upload-images.jianshu.io/upload_images/1258499-a979349fe66d80c3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
