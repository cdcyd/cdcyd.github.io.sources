---
title: OpenGL ES(五) 光照
date: 2016-10-14 16:40:00
tags: "OpenGL"
categories: "OpenGL ES"
---

在OpenGL ES中光照模型主要结构由3个元素组成：环境(Ambient)光照、漫反射(Diffuse)光照和镜面(Specular)光照
+ 环境光照：来自散落于我们周围的很多光源，这些来自四周的光源总会为物体的表面着色
+ 漫反射光照：漫反射光照是让物体产生视觉影响的主要光照，它特点是面向光源的一面比其他面会更亮
+ 镜面光照：镜面光照根据光的反射特性，让有光泽的物体出现亮点

在OpenGL中，我们会在自定义shader中，自己写这3种光照计算算法，但是在OpenGL ES，我们使用GLKit会简化很多，下面就是一个使用光照的简单例子：
```
-(void)setupGL{
    // 设置设备上下文
    GLKView *view = (GLKView *)self.view;
    view.context = [[EAGLContext alloc] initWithAPI:kEAGLRenderingAPIOpenGLES2];
    view.drawableDepthFormat = GLKViewDrawableDepthFormat24;
    [EAGLContext setCurrentContext:view.context];
    
    // case 1：设置环境光
    self.effect = [[GLKBaseEffect alloc] init];
    // 开启light0光照，light0默认是关闭的
    self.effect.light0.enabled = GL_TRUE;
    // 设置为绿色的环境光，如果这样运行Demo，可以看到的是绿色的球体
    self.effect.light0.ambientColor = GLKVector4Make(0.0f, 1.0f, 0.0f, 1.0f);

    // case 2: 添加散射光
    self.effect = [[GLKBaseEffect alloc] init];
    self.effect.light0.enabled = GL_TRUE;
    self.effect.light0.ambientColor = GLKVector4Make(0.0f, 1.0f, 0.0f, 1.0f);
    // 这时可以看到，面向光照的一面是红色的，而背向光照的一面是深绿色
    self.effect.light0.diffuseColor = GLKVector4Make(1.0f, 0.0f, 0.0f, 1.0f);

    // case 3：添加镜面光
    self.effect = [[GLKBaseEffect alloc] init];
    self.effect.light0.enabled = GL_TRUE;
    self.effect.light0.ambientColor = GLKVector4Make(0.0f, 1.0f, 0.0f, 1.0f);
    self.effect.light0.diffuseColor = GLKVector4Make(1.0f, 0.0f, 0.0f, 1.0f);
    // 这里需要注意，而在GLKit中材质的镜面光默认值是{0.0f, 0.0f, 0.0f, 1.0f}，这样设置光照的镜面光是没有效果的
    // 所以这里我们需要设置材质的镜面光
    self.effect.material.specularColor = GLKVector4Make(0.8f, 0.8f, 0.8f, 0.0f);
    // 材质的发光值，发光值越高，聚光效果越好
    self.effect.material.shininess = 32;
    // 设置光照的镜面光
    self.effect.light0.specularColor = GLKVector4Make(0.0f, 0.0f, 1.0f, 0.0f);

    // case 4：多光源
    self.effect = [[GLKBaseEffect alloc] init];
    // 材质镜面光
    self.effect.material.specularColor = GLKVector4Make(0.8f, 0.8f, 0.8f, 0.0f);
    self.effect.material.shininess = 32;
    // 第一个光源
    self.effect.light0.enabled = GL_TRUE;
    self.effect.light0.ambientColor = GLKVector4Make(0.0f, 1.0f, 0.0f, 1.0f);
    self.effect.light0.diffuseColor = GLKVector4Make(1.0f, 0.0f, 0.0f, 1.0f);
    self.effect.light0.specularColor = GLKVector4Make(0.0f, 0.0f, 1.0f, 0.0f);
    // 第二个光源
    self.effect.light1.enabled = GL_TRUE;
    self.effect.light1.position = GLKVector4Make(1.0f, 0.0f, 1.0f, 0.0f);
    self.effect.light1.ambientColor = GLKVector4Make(0.0f, 0.0f, 0.0f, 1.0f);
    self.effect.light1.diffuseColor = GLKVector4Make(0.0f, 1.0f, 1.0f, 1.0f);
    self.effect.light1.specularColor = GLKVector4Make(1.0f, 0.0f, 0.0f, 0.0f);
    
    关于材质
    在case 3中，我们设置了材质的镜面光，材质还有环境光和散射光，那么它们和光源的环境光、散射光和镜面光有什么不同呢？
    材质的这些光就是材质的属性，一个物体，它之所以会有颜色，是因为它反射了有颜色的光。例如一个蓝色的物体，当太阳光照到它时，它吸收了其他颜色的光，只反射了蓝色光。
    举个例子：
    如果我们设置材质镜面光为{0.0f, 0.0f, 0.8f, 0.0f}，
    此时，当材质受到镜面光照射时，它会反射0倍的红色，0倍的绿色，0.8倍的红色
    此时，再设置光照的镜面光为{1.0f, 1.0f, 0.0f, 0.0f}，那么将毫无镜面光效果，因为此时的材质根本不会反射红色和绿色。
    这也就解释了case 3中，我们为什么要设置材质的镜面光了，因为材质的镜面光默认都是0，我们无论怎么设置光照的镜面光，都是没有效果的
    我们也可以设置材质环境光和散射光，它们和镜面光同理

    glEnable(GL_DEPTH_TEST);
}
```

![case 1 截图](http://upload-images.jianshu.io/upload_images/1258499-0111f23c1b1892ed.PNG?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![case 2 截图](http://upload-images.jianshu.io/upload_images/1258499-a5892376652a3764.PNG?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![case 3 截图](http://upload-images.jianshu.io/upload_images/1258499-61d2f919fcf0748e.PNG?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![case 4 截图](http://upload-images.jianshu.io/upload_images/1258499-bf75b98027614c08.PNG?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

```
// 绘制3个球体的方法
-(void)glkView:(GLKView *)view drawInRect:(CGRect)rect{
    glClearColor(0xeb/255.f, 0xf5/255.f, 0xff/255.f, 1.0f);
    glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT);
    
    for (int i = 0 ; i < 3; i ++) {
        [self.effect prepareToDraw];
        
        float aspect = fabs(self.view.bounds.size.width / self.view.bounds.size.height);
        GLKMatrix4 projectionMatrix = GLKMatrix4MakePerspective(GLKMathDegreesToRadians(65.0f), aspect, 0.1f, 100.0f);
        self.effect.transform.projectionMatrix = projectionMatrix;
        self.effect.transform.modelviewMatrix = GLKMatrix4MakeTranslation(0.0f, 3*i-2, -(i+8));
        
        glGenBuffers(1, &_vertexID);
        glBindBuffer(GL_ARRAY_BUFFER, _vertexID);
        glBufferData(GL_ARRAY_BUFFER, sizeof(sphereVerts), sphereVerts, GL_STATIC_DRAW);
        
        glGenBuffers(1, &_normalID);
        glBindBuffer(GL_ARRAY_BUFFER, _normalID);
        glBufferData(GL_ARRAY_BUFFER, sizeof(sphereNormals), sphereNormals, GL_STATIC_DRAW);
        
        glEnableVertexAttribArray(GLKVertexAttribPosition);
        glVertexAttribPointer(GLKVertexAttribPosition, 3, GL_FLOAT, GL_FALSE, 3*sizeof(GLfloat), 0);
        
        glEnableVertexAttribArray(GLKVertexAttribNormal);
        glVertexAttribPointer(GLKVertexAttribNormal, 3, GL_FLOAT, GL_FALSE, 3*sizeof(GLfloat), 0);
        
        glDrawArrays(GL_TRIANGLES, 0, sphereNumVerts);
        
        glDeleteBuffers(1, &_vertexID);
        _vertexID = 0;
        glDeleteBuffers(1,&_normalID);
        _normalID = 0;
    }
}
```
```
// 删除缓存
- (void)dealloc
{
    [EAGLContext setCurrentContext:((GLKView *)self.view).context];
    if (_vertexID != 0){
        glDeleteBuffers (1,&_vertexID);
        _vertexID = 0;
    }
    if (_normalID != 0){
        glDeleteBuffers (1,&_normalID);
        _normalID = 0;
    }
    [EAGLContext setCurrentContext:nil];
}
```

Demo下载地址：https://github.com/cdcyd/CCOpenGLES