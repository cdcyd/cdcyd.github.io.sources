---
title: AlamofireImage 源码阅读
date: 2018-05-28 18:00:00
tags: "Image"
categories: "源码阅读"
---

在AlamofireImage中一共就只有5个类加一些扩展
```
// 错误处理类，继承自Error，主要有requestCancelled(请求取消)、imageSerializationFailed(请求失败)两种错误
AFIError
// 定义图片对象，主要用来适配mac(NSImage)和ios(UIImage)平台
Image
// 图片内存缓存对象
ImageCache
// 图片下载对象(下载基于Alamofire)
ImageDownloader
// 图片滤镜对象(CoreGraphics切圆角，CoreImage滤镜)
ImageFilter
```

## 一、图片加载过程
AlamofireImage中的扩展定义了很多快速对UI控件设置图片的方法，我挑其中一个来详解AlamofireImage是怎样将图片加载到视图上的
```
// 该方法是UIImageView的一个扩展方法，其它控件的扩展方法都差不多一样
public func af_setImage(
        withURLRequest urlRequest: URLRequestConvertible,
        placeholderImage: UIImage? = nil,
        filter: ImageFilter? = nil,
        progress: ImageDownloader.ProgressHandler? = nil,
        progressQueue: DispatchQueue = DispatchQueue.main,
        imageTransition: ImageTransition = .noTransition,
        runImageTransitionIfCached: Bool = false,
        completion: ((DataResponse<UIImage>) -> Void)? = nil)
    {
        /* 1.判断ImageView是否正在下载该url图片
         注：Alamofire通过runtime将正在下载图片的请求对象RequestReceipt绑定到af_activeRequestReceipt属性， 在请求完成后再将af_activeRequestReceipt置为nil，
         所以如果ImageView正在下载图片，而你此时再次请求下载图片时，它会进行判断，
         如果两次的url相同，则不需要再次下载了，所有这里会立即返回错误："取消了一次请求"
        */
        guard !isURLRequestURLEqualToActiveRequestURL(urlRequest) else {
            let error = AFIError.requestCancelled
            let response = DataResponse<UIImage>(request: nil, response: nil, data: nil, result: .failure(error))
            completion?(response)
            return
        }

        // 2.取消之前的所有请求
        af_cancelImageRequest()

        // 3.获取一个下载器
        let imageDownloader = af_imageDownloader ?? UIImageView.af_sharedImageDownloader

        // 4.获取下载器的缓存对象
        let imageCache = imageDownloader.imageCache

        // 5.如果存在缓存
        if  let request = urlRequest.urlRequest,
            let image = imageCache?.image(for: request, withIdentifier: filter?.identifier)
        {
            // 构建一个返回对象
            let response = DataResponse<UIImage>(request: request, response: nil, data: nil, result: .success(image))
            if runImageTransitionIfCached {
                // 如果有图片加载动画，加载图片并执行动画，并执行完成回调闭包
                let tinyDelay = DispatchTime.now() + Double(Int64(0.001 * Float(NSEC_PER_SEC))) / Double(NSEC_PER_SEC)
                DispatchQueue.main.asyncAfter(deadline: tinyDelay) {
                    self.run(imageTransition, with: image)
                    completion?(response)
                }
            } else {
                // 如果没有图片加载动画，直接加载图片，并执行完成回调闭包
                self.image = image
                completion?(response)
            }
            return
        }

        // 6.如果没有缓存，则先加载占位图
        if let placeholderImage = placeholderImage { self.image = placeholderImage }

        // 生成唯一的下载标识符，以检查下载在请求中是否发生了更改
        let downloadID = UUID().uuidString

        // 7.开始下载图片
        let requestReceipt = imageDownloader.download(
            urlRequest,
            receiptID: downloadID,
            filter: filter,
            progress: progress,
            progressQueue: progressQueue,
            completion: { [weak self] response in
                // 如果ImageView当前下载图片的url和返回的url相同，并且下载的标识符也相同，说明已经下载过了，所以这里直接返回完成，不需要再做其它了
                guard let strongSelf = self,
                          strongSelf.isURLRequestURLEqualToActiveRequestURL(response.request) &&
                          strongSelf.af_activeRequestReceipt?.receiptID == downloadID else {
                    completion?(response)
                    return
                }

                // 8.下载图片成功，加载图片
                if let image = response.result.value {
                    strongSelf.run(imageTransition, with: image)
                }

                // 9.清除当前的下载票据，回调加载下载完成
                strongSelf.af_activeRequestReceipt = nil
                completion?(response)
            }
        )

        // 存储ImageView正在请求的信息
        af_activeRequestReceipt = requestReceipt
    }

```
下面是下载类核心方法和注解
```
@discardableResult
    open func download(
        _ urlRequest: URLRequestConvertible,
        receiptID: String = UUID().uuidString,
        filter: ImageFilter? = nil,
        progress: ProgressHandler? = nil,
        progressQueue: DispatchQueue = DispatchQueue.main,
        completion: CompletionHandler?)
        -> RequestReceipt?
    {
        var request: DataRequest!
        
        // 异步加载图片
        synchronizationQueue.sync {
            // 再次判断该请求是否正在请求，如果是，则在responseHandlers属性中添加本次的回调闭包(多个view同时加载同一张图片的情况)
            // 注：ImageDownloader在responseHandlers属性中，存储正在下载的请求，以防止相同的请求多次发出，
            // 这样不仅浪费流量还会影响加载速度，如果有多个相同的请求时，我们只会发出一个，在完成后一起回调
            let urlID = ImageDownloader.urlIdentifier(for: urlRequest)
            if let responseHandler = self.responseHandlers[urlID] {
                responseHandler.operations.append((receiptID: receiptID, filter: filter, completion: completion))
                request = responseHandler.request
                return
            }

            // (内存缓存)如果允许缓存，再次尝试从缓存加载图像
            if let request = urlRequest.urlRequest {
                switch request.cachePolicy {
                case .useProtocolCachePolicy, .returnCacheDataElseLoad, .returnCacheDataDontLoad:
                    if let image = self.imageCache?.image(for: request, withIdentifier: filter?.identifier) {
                        DispatchQueue.main.async {
                            let response = DataResponse<Image>(
                                request: urlRequest.urlRequest,
                                response: nil,
                                data: nil,
                                result: .success(image)
                            )
                            completion?(response)
                        }
                        return
                    }
                default:
                    break
                }
            }

            // 创建下载器
            request = self.sessionManager.request(urlRequest)
            if let credential = self.credential {
                request.authenticate(usingCredential: credential)
            }

            request.validate()
            if let progress = progress {
                request.downloadProgress(queue: progressQueue, closure: progress)
            }

            // 生成唯一的下载标识符，以检查下载过程中请求是否发生了更改
            let handlerID = UUID().uuidString

            // 开始请求(这其中默认会去获取NSURLCache的缓存，内存缓存和磁盘缓存均有)
            request.response(queue: self.responseQueue, responseSerializer: imageResponseSerializer, completionHandler: { [weak self] response in
                    guard let strongSelf = self, let request = response.request else { return }

                    defer {
                        // 减少当前需要下载的任务数
                        strongSelf.safelyDecrementActiveRequestCount()
                        // 如果还有，则开始下一个下载
                        strongSelf.safelyStartNextRequestIfNecessary()
                    }

                    // 将该请求从正在下载任务中移除
                    let handler = strongSelf.safelyFetchResponseHandler(withURLIdentifier: urlID)
                    guard handler?.handlerID == handlerID else { return }
                    guard let responseHandler = strongSelf.safelyRemoveResponseHandler(withURLIdentifier: urlID) else {
                        return
                    }

                    switch response.result {
                    case .success(let image):
                        var filteredImages: [String: Image] = [:]

                        // 下载完成
                        for (_, filter, completion) in responseHandler.operations {
                            var filteredImage: Image

                            if let filter = filter {
                                if let alreadyFilteredImage = filteredImages[filter.identifier] {
                                    filteredImage = alreadyFilteredImage
                                } else {
                                    filteredImage = filter.filter(image)
                                    filteredImages[filter.identifier] = filteredImage
                                }
                            } else {
                                filteredImage = image
                            }

                            // 内存缓存图片
                            strongSelf.imageCache?.add(filteredImage, for: request, withIdentifier: filter?.identifier)

                            // 返回Image
                            DispatchQueue.main.async {
                                let response = DataResponse<Image>(
                                    request: response.request,
                                    response: response.response,
                                    data: response.data,
                                    result: .success(filteredImage),
                                    timeline: response.timeline
                                )
                                completion?(response)
                            }
                        }
                    case .failure:
                        // 下载失败
                        for (_, _, completion) in responseHandler.operations {
                            DispatchQueue.main.async { completion?(response) }
                        }
                    }
                }
            )

            // 存储正在下载的任务
            let responseHandler = ResponseHandler(
                request: request,
                handlerID: handlerID,
                receiptID: receiptID,
                filter: filter,
                completion: completion
            )
            self.responseHandlers[urlID] = responseHandler

            // 开始任务或者加载到任务队列中
            if self.isActiveRequestCountBelowMaximumLimit() {
                self.start(request)
            } else {
                self.enqueue(request)
            }
        }
        if let request = request {
            return RequestReceipt(request: request, receiptID: receiptID)
        }
        return nil
    }
```
总结下载过程
1.在内存缓存对象(ImageCache)中获取缓存，如果有则返回图片
2.在NSURLCache中获取缓存(内存缓存+磁盘缓存)，如果有则返回图片
3.开始网络下载图片，成功后返回图片
4.缓存图片
5.检查是否使用滤镜、加载动画等加载图片

## 二、缓存实现
1.ImageCache，实际上它只是一个协议，真正缓存的对象是CachedImage
```
class CachedImage {
    // 图片
    let image: Image
    // 标识符
    let identifier: String
    // 图片大小
    let totalBytes: UInt64
    // 最后修改时间
    var lastAccessDate: Date 
}
```
在ImageCache协议中有如下几个方法
```
/// 通过标识符添加一个缓存图片
func add(_ image: Image, withIdentifier identifier: String)

/// 通过标识符移除一个图片
func removeImage(withIdentifier identifier: String) -> Bool

/// 移除所有图片
func removeAllImages() -> Bool

/// 通过标识符获取图片
func image(withIdentifier identifier: String) -> Image?
```
实际对图片存储的类是AutoPurgingImageCache，它实现了ImageCache协议，下再来看一下它的一些属性
```
// 最大内存容量
open let memoryCapacity: UInt64
// 当内存容量达到最大值，清除后的剩余内存(当内存达到最大值时：清除部分 = memoryCapacity - preferredMemoryUsageAfterPurge)
open let preferredMemoryUsageAfterPurge: UInt64
// 操作线程
private let synchronizationQueue: DispatchQueue
// 缓存(说明AlamofireImage的图片缓存是一个字典)
private var cachedImages: [String: CachedImage]
// 当前内存
private var currentMemoryUsage: UInt64
```
下面是添加缓存的核心函数
```
open func add(_ image: Image, withIdentifier identifier: String) {
        synchronizationQueue.async(flags: [.barrier]) {
            // 缓存图片，并计算当前的内存缓存大小
            let cachedImage = CachedImage(image, identifier: identifier)
            if let previousCachedImage = self.cachedImages[identifier] {
                self.currentMemoryUsage -= previousCachedImage.totalBytes
            }
            self.cachedImages[identifier] = cachedImage
            self.currentMemoryUsage += cachedImage.totalBytes
        }
        // 这里的barrier，保证了当上面的图片缓存完成之后才会执行下面的代码
        synchronizationQueue.async(flags: [.barrier]) {
            // 当前缓存大于最大缓存时
            if self.currentMemoryUsage > self.memoryCapacity {
                // 计算需要清除的缓存大小
                let bytesToPurge = self.currentMemoryUsage - self.preferredMemoryUsageAfterPurge
                // 对图片进行时间排序，我们需要清除较早的一些图片
                var sortedImages = self.cachedImages.map { $1 }
                sortedImages.sort {
                    let date1 = $0.lastAccessDate
                    let date2 = $1.lastAccessDate
                    return date1.timeIntervalSince(date2) < 0.0
                }
                // 开始清除缓存
                var bytesPurged = UInt64(0)
                for cachedImage in sortedImages {
                    self.cachedImages.removeValue(forKey: cachedImage.identifier)
                    bytesPurged += cachedImage.totalBytes
                    if bytesPurged >= bytesToPurge {
                        break
                    }
                }
                // 清除缓存完成，设置当前缓存的大小
                self.currentMemoryUsage -= bytesPurged
            }
        }
    }
```
2.NSURLCache，它我这里就不具体讲了，有兴趣的可以看这篇文章   http://nshipster.cn/nsurlcache/

## 三、加载动画和滤镜
对于这一部分内容，我自己也没有使用过，所以下面只贴出源码加注释，有兴趣的读者可以自己去研究

1.动画
```
// 加载动画(是用UIView的动画来实现的)
public func run(_ imageTransition: ImageTransition, with image: Image) {
    UIView.transition(
        with: self,
        duration: imageTransition.duration,
        options: imageTransition.animationOptions,
        animations: { imageTransition.animations(self, image) },
        completion: imageTransition.completion
    )
}
public enum ImageTransition {
    // 所有可选的动画类型
    // 没有动画
    case noTransition
    // 淡入淡出
    case crossDissolve(TimeInterval)
    // 向下翻页
    case curlDown(TimeInterval)
    // 向上翻页
    case curlUp(TimeInterval)
    // 从下、左、右、上翻出
    case flipFromBottom(TimeInterval)
    case flipFromLeft(TimeInterval)
    case flipFromRight(TimeInterval)
    case flipFromTop(TimeInterval)
    // 自定义动画
    case custom(
        duration: TimeInterval,
        animationOptions: UIViewAnimationOptions,
        animations: (UIImageView, Image) -> Void,
        completion: ((Bool) -> Void)?
    )
}
```

2.滤镜
```
// 将图片变化到指定大小
public struct ScaledToSizeFilter: ImageFilter, Sizable {
    /// 变换后的大小
    public let size: CGSize
    public init(size: CGSize) {
        self.size = size
    }
    /// 实现函数
    public var filter: (Image) -> Image {
        return { image in
            return image.af_imageScaled(to: self.size)
        }
    }
}

/// 拉伸适应
public struct AspectScaledToFitSizeFilter: ImageFilter, Sizable {
    public let size: CGSize
    public init(size: CGSize) {
        self.size = size
    }
    /// 实现函数
    public var filter: (Image) -> Image {
        return { image in
            return image.af_imageAspectScaled(toFit: self.size)
        }
    }
}

/// 拉伸铺满
public struct AspectScaledToFillSizeFilter: ImageFilter, Sizable {
    public let size: CGSize
    public init(size: CGSize) {
        self.size = size
    }
    /// 实现函数
    public var filter: (Image) -> Image {
        return { image in
            return image.af_imageAspectScaled(toFill: self.size)
        }
    }
}

/// 圆角
public struct RoundedCornersFilter: ImageFilter, Roundable {
    /// 圆角大小 
    public let radius: CGFloat
    /// 圆角的大小需不需要进行缩放radius / scale
    public let divideRadiusByImageScale: Bool
    public init(radius: CGFloat, divideRadiusByImageScale: Bool = false) {
        self.radius = radius
        self.divideRadiusByImageScale = divideRadiusByImageScale
    }
    /// 实现函数
    public var filter: (Image) -> Image {
        return { image in
            return image.af_imageRounded(
                withCornerRadius: self.radius,
                divideRadiusByImageScale: self.divideRadiusByImageScale
            )
        }
    }
}

/// 圆形图片
public struct CircleFilter: ImageFilter {
    public init() {}
    /// 实现函数
    public var filter: (Image) -> Image {
        return { image in
            return image.af_imageRoundedIntoCircle()
        }
    }
}

// CoreImage 滤镜
public protocol CoreImageFilter: ImageFilter {
    /// 滤镜名称
    var filterName: String { get }
    /// 滤镜参数
    var parameters: [String: Any] { get }
}

/// 高斯模糊滤镜
public struct BlurFilter: ImageFilter, CoreImageFilter {
    /// 高斯滤镜名称
    public let filterName = "CIGaussianBlur"
    /// 参数
    public let parameters: [String: Any]
    public init(blurRadius: UInt = 10) {
        self.parameters = ["inputRadius": blurRadius]
    }
}
```
3.CoreImage 滤镜具体实现的核心函数
```
extension UIImage {
    public func af_imageFiltered(withCoreImageFilter name: String, parameters: [String: Any]? = nil) -> UIImage? {
        var image: CoreImage.CIImage? = ciImage

        if image == nil, let CGImage = self.cgImage {
            image = CoreImage.CIImage(cgImage: CGImage)
        }

        guard let coreImage = image else { return nil }

        let context = CIContext(options: [kCIContextPriorityRequestLow: true])

        var parameters: [String: Any] = parameters ?? [:]
        parameters[kCIInputImageKey] = coreImage

        guard let filter = CIFilter(name: name, withInputParameters: parameters) else { return nil }
        guard let outputImage = filter.outputImage else { return nil }

        let cgImageRef = context.createCGImage(outputImage, from: outputImage.extent)

        return UIImage(cgImage: cgImageRef!, scale: scale, orientation: imageOrientation)
    }
}
```