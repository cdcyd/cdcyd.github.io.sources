---
title: OpenGL ES(三) 纹理
date: 2016-08-15 13:35:00
tags: "OpenGL"
categories: "OpenGL ES"
---

纹理是一种应用到OpenGL绘图场景中三角形上的图像数据，它通过经过过滤纹理单元填充到实心区域。

下面是OpenGL ES载入一个简单纹理的例子

 ```
-(void)setupGL{
    // 创建设备上下文，用OpenGL ES 2.0的API
    GLKView *view = (GLKView *)self.view;
    view.context = [[EAGLContext alloc] initWithAPI:kEAGLRenderingAPIOpenGLES2];
    // GLKView的深度缓存
    view.drawableDepthFormat = GLKViewDrawableDepthFormat24;
    [EAGLContext setCurrentContext:view.context];
    
    // 创建GLKBaseEffect
    self.baseEffect = [[GLKBaseEffect alloc]init];
    self.baseEffect.useConstantColor = GL_TRUE;
    self.baseEffect.constantColor = GLKVector4Make(1.0f, 1.0f, 1.0f, 1.0f);// 设置三角形颜色(注：如果开启光照，这里的颜色将会失效)

    // 顶点数据(前3列是顶点数据，一共6个顶点构成一个矩形，后2列是纹理坐标，这里需要注意纹理坐标原点和OpenGL ES的绘图坐标的原点是不一样的
    // OpenGL ES的绘图坐标的原点在屏幕中间
    // 纹理坐标分为两种情况：在使用GLKit时，纹理坐标在右上角；使用shader绘图时，原点在左下角)
    GLfloat vertexs[] = {
        -0.5f, 0.5f, 0.0f,    0.0f,0.0f,
         0.5f, 0.5f, 0.0f,    1.0f,0.0f,
        -0.5f,-0.5f, 0.0f,    0.0f,1.0f,
        -0.5f,-0.5f, 0.0f,    0.0f,1.0f,
         0.5f,-0.5f, 0.0f,    1.0f,1.0f,
         0.5f, 0.5f, 0.0f,    1.0f,0.0f
    };
    
    // 载入顶点数据
    glGenBuffers(1, &_VBO);
    glBindBuffer(GL_ARRAY_BUFFER, _VBO);
    glBufferData(GL_ARRAY_BUFFER, sizeof(vertexs), vertexs, GL_STATIC_DRAW);
    
    // 载入一个纹理
    CGImageRef imageRef = [[UIImage imageNamed:@"texture256x256.jpg"] CGImage];
    GLKTextureInfo *textureInfo = [GLKTextureLoader 
                                   textureWithCGImage:imageRef 
                                   options:nil 
                                   error:NULL];
    self.baseEffect.texture2d0.name = textureInfo.name;
    self.baseEffect.texture2d0.target = textureInfo.target;
    
    // 变换(OpenGL坐标中，以屏幕中间为原点，向右到屏幕边缘为x轴的0~1，向上为y轴的0~1，向屏幕外为z轴的正方向
    // 由于我们的设备是高大于宽的，所有y轴0.5大于x轴0.5，所以上面的顶点数据的输出是一个长方形，但是我们的期望是输出一个正方形，下面的变换就是为了解决这个问题)
    float aspect = fabs(self.view.bounds.size.width / self.view.bounds.size.height);
    // 创建透视矩阵(参数1：视角，参数2：视图宽高比，参数3：近视点，参数4：远视点)
    GLKMatrix4 projectionMatrix = GLKMatrix4MakePerspective(GLKMathDegreesToRadians(65.0f), aspect, 0.1f, 100.0f);
    self.baseEffect.transform.projectionMatrix = projectionMatrix; // 透视变换
    self.baseEffect.transform.modelviewMatrix = GLKMatrix4MakeTranslation(0.0f, 0.0f, -2.0f); // 模型变换(沿着z轴移动2个单位)
    
    // 启用顶点缓存数据
    glBindBuffer(GL_ARRAY_BUFFER, _VBO);
    glEnableVertexAttribArray(GLKVertexAttribPosition);
    glVertexAttribPointer(GLKVertexAttribPosition, 3, GL_FLOAT, GL_FALSE, 5 * sizeof(GLfloat), BUFFER_OFFSET(0));

    // 启用纹理缓存数据
    glBindBuffer(GL_ARRAY_BUFFER, _VBO);
    glEnableVertexAttribArray(GLKVertexAttribTexCoord0);
    glVertexAttribPointer(GLKVertexAttribTexCoord0, 2, GL_FLOAT, GL_FALSE, 5 * sizeof(GLfloat), BUFFER_OFFSET(3*sizeof(GLfloat)));
}
 ```

```
// 绘图函数
-(void)glkView:(GLKView *)view drawInRect:(CGRect)rect{
    
    glClearColor(0xeb/255.f, 0xf5/255.f, 0xff/255.f, 1.0f);
    glClear(GL_COLOR_BUFFER_BIT);
    
    [self.baseEffect prepareToDraw];
    glDrawArrays(GL_TRIANGLES, 0, 6);
}
```

```
// 清除缓存
- (void)dealloc
{
    [EAGLContext setCurrentContext:((GLKView *)self.view).context];
    if (_VBO != 0){
        glDeleteBuffers (1,&_VBO);
        _VBO = 0;
    }
    [EAGLContext setCurrentContext:nil];
}
```

Demo下载地址：https://github.com/cdcyd/CCOpenGLES