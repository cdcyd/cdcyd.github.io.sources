---
title: OpenGL ES(一) 初识
date: 2016-02-02 17:08:00
tags: "OpenGL"
categories: "OpenGL ES"
---

1.OpenGL
OpenGL：图形硬件的一种软件接口,它是一个3D图形和模型库，我们可以使用OpenGL来创建实时的3D图形或模型，并且它不仅有出色的视觉质量，还有它的效率远高于光线追踪器或软件渲染引擎。

2.OpenGL ES
OpenGL ES与OpenGL非常相似，因为OpenGL ES的规范是基于OpenGL开发的，专门为移动设备的3D渲染提供渲染接口，可以看做精简版的OpenGL。

3.版本发展

![OpenGL ES 与相关OpenGL版本](http://upload-images.jianshu.io/upload_images/1258499-e37895d0d3c9485d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

4.OpenGL ES绘制一个 Core Animation 层的过程
   + 创建设备上下文
   + 创建GLKBaseEffect(苹果封装的可以简化OpenGL绘制操作的类)
   + 渲染(通过缓存绘图)
       - 生成控制缓存的标示符(Generate) - **_glGenBuffers()_**
       - 让OpenGL ES知道接下来的运算会使用一个缓存(Bind) - **_glBindBuffer()_**
       - 让OpenGL ES分配连续的内存并初始化缓存(Buffer Data) - **_glBufferData()_**
       - 告诉OpenGL ES接下来渲染该缓存(Enable) -  **_glEnableVertexAttribArray()_**
       - 告诉OpenGL ES指定的顶点属性(Set Pointers) -  **_glVetexAttribPointer()_**
       - 绘图(Draw) - **_glDrawArrays()_**
   + 删除缓存并释放相关的资源(Delete) - **_glDeleteBuffers()_**

5.Demo下载地址：https://github.com/cdcyd/CCOpenGLES