---
title: CoreImage滤镜效果预览
date: 2018-03-21 14:54:00
tags: "CoreImage"
categories: "Swift"
---

闲来无事写的 Core Image 里的滤镜效果的Demo，目前 Core Image 有100多种滤镜效果，Demo实现了其中大概一半的效果，实在太多了，后面有时间再补全，有兴趣的可以下载下来看一下(最好用真机查看，模拟器实在太卡了)
Demo地址： https://github.com/cdcyd/CCFilters
滤镜的核心实现代码只需三行，如下
```
func filter(name: String, parameters: [String:Any]) -> UIImage? {
    guard let image = self.cgImage else {
        return nil
    }

    #if DEBUG
    let filter = CIFilter(name: name)
    print(filter?.attributes as Any)
    #endif

    // 输入CIImage
    let input = CIImage(cgImage: image)

    // 输出CIImage
    let output = input.applyingFilter(name, parameters: parameters)

    // 渲染图片(CIImage->CGImage)
    guard let cgimage = CIContext(options: nil).createCGImage(output, from: input.extent) else {
        return nil
    }
    return UIImage(cgImage: cgimage)
}
```
