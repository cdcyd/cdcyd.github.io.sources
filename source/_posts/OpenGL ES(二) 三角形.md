---
title: OpenGL ES(二) 三角形
date: 2016-08-09 14:55:00
tags: "OpenGL"
categories: "OpenGL ES"
---

相比于OpenGL绘图来说，OpenGL ES要简单很多，因为苹果公司给我们封装了工具类GLKBaseEffect，下面是一个简单的绘制三角形的例子：

```
-(void)setupGL{
    // 创建设备上下文，用OpenGL ES 2.0的API
    GLKView *view = (GLKView *)self.view;
    view.context = [[EAGLContext alloc] initWithAPI:kEAGLRenderingAPIOpenGLES2];
    // GLKView的深度缓存
    view.drawableDepthFormat = GLKViewDrawableDepthFormat24;
    [EAGLContext setCurrentContext:view.context];
    
    // 创建GLKBaseEffect
    self.effect = [[GLKBaseEffect alloc]init];
    self.effect.useConstantColor = GL_TRUE; 
    self.effect.constantColor = GLKVector4Make(1.0f, 0.5f, 0.2f, 1.0); // 设置三角形颜色(注：如果开启光照，这里的颜色将会失效)
    
    // 顶点数据
    GLfloat vertices[] = {
        -0.5f, -0.5f, 0.0f,
         0.5f, -0.5f, 0.0f,
        -0.5f,  0.5f, 0.0f
    };

    // 生成顶点数据缓存标示符(参数1：指定生产缓存标示符的数量，参数2：指针指向标示符的内存保存位置)
    glGenBuffers(1, &vertexBufferID);
    // 绑定缓存，将指定标示符的缓存用到当前缓存，同一时刻只能绑定一个(参数1：指定绑定缓存的类型，参数2：缓存的标示符)
    glBindBuffer(GL_ARRAY_BUFFER, vertexBufferID);
    // 初始化一个缓存，并将数据复制到缓存中
    glBufferData(GL_ARRAY_BUFFER, sizeof(vertices), vertices, GL_STATIC_DRAW);
    
    // 启动顶点缓存渲染操作
    glEnableVertexAttribArray(GLKVertexAttribPosition);
    // 链接顶点保存的数据(参数1：当前每个顶点的位置信息，参数2：每个顶点有3个数据，参数3：每个数据是一个浮点值，参数4：小数点固定数据是否可以改变，参数5：每个顶点保存需要多少字节，参数6：访问数据的偏移值)
    glVertexAttribPointer(GLKVertexAttribPosition, 3, GL_FLOAT, GL_FALSE, 3*sizeof(GLfloat), NULL);
}
```
```
// 绘图函数
-(void)glkView:(GLKView *)view drawInRect:(CGRect)rect
{
    // 清除颜色(设置背景颜色)
    glClearColor(0xeb/255.f, 0xf5/255.f, 0xff/255.f, 1.0f);
    // 清除颜色帧缓存
    glClear(GL_COLOR_BUFFER_BIT); 
  
    // 准备绘图
    [self.effect prepareToDraw]; 
    // 执行绘图(参数1：告诉GPU怎么处理绑定在顶点缓存内的顶点数据，参数2：指定需要渲染的第一个顶点，参数3：需要渲染的顶点数量)
    glDrawArrays(GL_TRIANGLES, 0, 3);
}
```
```
// 删除不需要的顶点缓存和上下文
-(void)dealloc
{
    GLKView *view = (GLKView *)self.view;
    [EAGLContext setCurrentContext:view.context];
    if (0 != vertexBufferID){
        glDeleteBuffers (1,&vertexBufferID);
        vertexBufferID = 0;
    }
    [EAGLContext setCurrentContext:nil];
}
```

Demo下载地址：https://github.com/cdcyd/CCOpenGLES