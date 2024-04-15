---
title: Core系列框架介绍(一)
date: 2017-12-25 17:22:00
tags: "Foundation"
categories: "Objective-C"
---

### 图层、图像相关框架：CoreGraphics(Quartz2D)、QuartzCore(CoreAnimation)、CoreImage、CoreText

<!-- more -->

1.CoreGraphics(Quartz2D)
```
import Darwin
import CoreGraphics
import CoreGraphics.CGBase
// 常用对象
import CoreGraphics.CGFunction
import CoreGraphics.CGImage  // 图片
import CoreGraphics.CGColor  // 颜色
import CoreGraphics.CGLayer  // 图层
import CoreGraphics.CGFont   // 字体
import CoreGraphics.CGPath   // 路径
import CoreGraphics.CGError
// CGPoint、CGSize、CGRect以及相关几何运算函数的定义
import CoreGraphics.CGGeometry
// 渐变
import CoreGraphics.CGGradient
import CoreGraphics.CGShading
// 变换
import CoreGraphics.CGAffineTransform
// 绘图、图像I/O相关
import CoreGraphics.CGContext
import CoreGraphics.CGBitmapContext
import CoreGraphics.CGPattern
import CoreGraphics.CGColorConversionInfo
import CoreGraphics.CGColorSpace
import CoreGraphics.CGDataConsumer
import CoreGraphics.CGDataProvider
// PDF文档创建、显示和解析相关
import CoreGraphics.CGPDFArray
import CoreGraphics.CGPDFContentStream
import CoreGraphics.CGPDFContext
import CoreGraphics.CGPDFDictionary
import CoreGraphics.CGPDFDocument
import CoreGraphics.CGPDFObject
import CoreGraphics.CGPDFOperatorTable
import CoreGraphics.CGPDFPage
import CoreGraphics.CGPDFScanner
import CoreGraphics.CGPDFStream
import CoreGraphics.CGPDFString
```
CoreGraphics，也称为Quartz2D，基于Darwin，它是一个2D绘图引擎，主要处理路径的绘制、抗锯齿、渐变、图像、颜色、PDF文档等
* 定义了CGPath、CGImage等常用的对象
* 定义了CGPoint、CGSize、CGRect等常用的数据结构并提供了相关的几何运算函数，
* 定义了CGLayer并提供了渐变和变换矩阵的接口
* 提供了绘图接口(CGContext)
* 提供了对图像I/O相关操作接口
* 提供了对PDF操作的接口

**所以CoreGraphics是系统绘制界面、图像、动画的基础框架**

2.QuartzCore(CoreAnimation)
```
import Foundation
import QuartzCore.CoreAnimation
import QuartzCore
// 动画(属性动画、关键帧动画等)
import QuartzCore.CABase
import QuartzCore.CAAnimation
// 几何变换相关
import QuartzCore.CATransaction
import QuartzCore.CATransform3D
import QuartzCore.CATransformLayer
// 时间相关
import QuartzCore.CADisplayLink
import QuartzCore.CAValueFunction
import QuartzCore.CAMediaTiming
import QuartzCore.CAMediaTimingFunction
// 特殊图层
import QuartzCore.CALayer
import QuartzCore.CAEAGLLayer         // OpenGL ES 绘图 图层
import QuartzCore.CAEmitterCell       // 粒子特效 Cell
import QuartzCore.CAEmitterLayer      // 粒子特效 图层
import QuartzCore.CAGradientLayer     // 渐变 图层
import QuartzCore.CAReplicatorLayer   // 复制 图层
import QuartzCore.CAScrollLayer       // 滚动 图层
import QuartzCore.CAShapeLayer        // 阴影 图层
import QuartzCore.CATextLayer         // 文本 图层
import QuartzCore.CATiledLayer        // 大图加载 图层
```
QuartzCore和CoreAnimation实际上可以看作同一个框架，它们互相引用，它们基于Metal和CoreGraphics，主要用于图形渲染和动画
* 提供了动画接口(属性动画、关键帧动画、组动画等)
* 提供了几何变换接口，是对CoreGraphics的CGAffineTransform进一步封装
* 封装了CALayer，它是使视图呈现出来的基础类
* 封装了一些特殊用途的图层Layer(如粒子特效CAEmitterLayer、渐变CAGradientLayer)等

3.CoreImage
```
// 上下文
import CoreImage.CIContext
// 检测
import CoreImage.CIDetector
// 特征
import CoreImage.CIFeature
// 滤镜
import CoreImage.CIFilter
// 条码、二维码
import CoreImage.CIBarcodeDescriptor
// 自定义滤镜相关
import CoreImage.CIFilterConstructor
import CoreImage.CIFilterShape
import CoreImage.CIColor
import CoreImage.CIImage
import CoreImage.CIVector
import CoreImage.CIImageAccumulator
import CoreImage.CIImageProcessor
import CoreImage.CIImageProvider
import CoreImage.CIKernel
import CoreImage.CIRAWFilter
import CoreImage.CIRenderDestination
import CoreImage.CISampler
import CoreImage.CoreImageDefines
import CoreImage
import Foundation
```
CoreImage是一个图像处理框架，为静态和视频图像提供接近实时的处理，CoreImage提供如下功能
* 滤镜：内置多个图像滤镜
* 滤镜图表：是一个链接在一起的滤镜网络 ，使得一个滤镜的输出可以是另一个滤镜的输入，以达到创建自定义滤镜的效果
* 特征检测

4.CoreText
```
import CoreText.CTDefines
import CoreText.CTFont
import CoreText.CTFontCollection
import CoreText.CTFontDescriptor
import CoreText.CTFontManager
import CoreText.CTFontManagerErrors
import CoreText.CTFontTraits
import CoreText.CTFrame
import CoreText.CTFramesetter
import CoreText.CTGlyphInfo
import CoreText.CTLine
import CoreText.CTParagraphStyle
import CoreText.CTRubyAnnotation
import CoreText.CTRun
import CoreText.CTRunDelegate
import CoreText.CTStringAttributes
import CoreText.CTTextTab
import CoreText.CTTypesetter
import CoreText.SFNTLayoutTypes
import CoreText.SFNTTypes
```
CoreText是一种文本处理技术，它基于CoreGraphics，主要实现文字的自定义排版
