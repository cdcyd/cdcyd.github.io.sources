---
title: OpenGL ES(四) 变换
date: 2016-10-11 16:41:00
tags: "OpenGL"
categories: "OpenGL ES"
---

基本变换：平移(translation)、旋转(roration)、缩放(scale)、透视(perspective)，这4个基本变换可以单独使用，也可以组合使用(两个基本变换可以使用矩阵乘法组合起来)
**_注意：当使用组合变换时，顺序很重要，例如平移和旋转组合，先平移和先旋转会得到两个完全不不同的结果_**

所有的基础变换矩阵，都可以通过GLKit/GLKMatrix4.h里的函数构建
+ 平移
// 返回一个平移矩阵：tx ty tz 分别是在x y z 轴的移动距离，
GLKMatrix4MakeTranslation(float tx, float ty, float tz)

+ 旋转
// 返回一个旋转矩阵
// radians是旋转角度，它接受一个弧度值，可以用GLKMathDegreesToRadians(30)，将角度转换为弧度
// 后面的x y z组成一个向量，顶点将围绕这个向量做旋转(如{1.0，0.0，0.0}，将会绕x轴做旋转)
GLKMatrix4MakeRotation(float radians, float x, float y, float z)

+ 缩放
// 返回一个缩放矩阵：sx sy sz 分别是x y z轴方向上的缩放倍数
GLKMatrix4MakeScale(float sx, float sy, float sz)

+ 透视
  视域(viewing volume)：在OpenGL中，使用视域来决定那些元素将会显示，如果元素在视域外，那么它将会被丢弃，也不会显示，如果在视域内，元素才会被显示
   投影(prohection):投影分为正射投影和透视投影，我们可以通过它来设置投影矩阵来设置视域，在OpenGL中，默认的投影矩阵是一个立方体，即x y z 分别是-1.0~1.0的距离，如果超出该区域，将不会被显示。
    - 正射投影(orthographic projection)：GLKMatrix4MakeOrtho(float left,  float righ,  float bottom, float top, float nearZ, float farZ)，该函数返回一个正射投影的矩阵，它定义了一个由 left、right、bottom、top、near、far 所界定的一个矩形视域。此时，视点与每个位置之间的距离对于投影将毫无影响。
    - 透视投影(perspective projection)：GLKMatrix4MakeFrustum(float left, float right,float bottom, float top, float nearZ, float farZ)，该函数返回一个透视投影的矩阵，它定义了一个由 left、right、bottom、top、near、far 所界定的一个平截头体(椎体切去顶端之后的形状)视域。此时，视点与每个位置之间的距离越远，对象越小。
  
  // GLKMatrix4.h为我们提供了一个快速创建透视矩阵的函数
  // fovyRadians是视角，它接受一个弧度值，可以用GLKMathDegreesToRadians(30)，将角度转换为弧度
  // aspect视图宽高比，nearZ近视点，farZ远视点
  GLKMatrix4MakePerspective(float fovyRadians, float aspect, float nearZ, float farZ)

projectionMatrix 和 modelviewMatrix
当我们构建好了变换矩阵之后怎么传递的OpenGL呢？GLKBaseEffect有一个transform的属性，其中有两个矩阵分别是projectionMatrix和modelviewMatrix
projectionMatrix：投影矩阵，下面就是设置一个正投影的代码
```
// 伪代码
const GLFloat aspect = view.width/view.hight；
baseEffect.transform.projectionMatrix = GLKMatrix4MakeOrtho(-0.5 * aspect,
                                                            0.5 * aspect,
                                                            -0.5,
                                                            0.5,
                                                            -50,
                                                            50);
```
modelviewMatrix：模型矩阵，下面是设置一个沿z轴平移的矩阵
```
// 伪代码
baseEffect.transform.modelviewMatrix = GLKMatrix4MakeTranslation(0.0f, 0.0f, -3.0f);
```

Demo下载地址：https://github.com/cdcyd/CCOpenGLES