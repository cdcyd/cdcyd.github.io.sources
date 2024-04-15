---
title: SDWebImage 源码阅读(缓存)
date: 2018-05-25 18:00:00
tags: "Image"
categories: "源码阅读"
---

在 SDWebImage 中，设计了两种缓存
1.`SDMemoryCache`：它继承自 `NSCache` 用来实现内存缓存
2.`NSFileManager`：使用文件的方式来实现磁盘缓存

+ 先来看一下 `SDImageCache` 的内存缓存的实现
```
@interface SDMemoryCache <KeyType, ObjectType> ()
@property (nonatomic, strong, nonnull) NSMapTable<KeyType, ObjectType> *weakCache; 
@property (nonatomic, strong, nonnull) dispatch_semaphore_t weakCacheLock; 
@end
```
1. weakCache：它是一个NSMapTable的对象，那么它和字典都有哪些区别呢？a> 它是可变的；b> 可以在添加value的时候对value进行复制；c> 可以通过弱引用来持有keys和values，所以当key或者value被deallocated的时候，所存储的实体也会被移除；
2. weakCacheLock：它是一个锁，用来保证对weakCache操作时的线程安全，所以在对SDWebImage的缓存研究时，我们可以忽略它
3. SDMemoryCache：它自己是NSCache，也会对图片进行内存缓存，并且它还是线程安全的

> 问题：既然NSCache已经可以实现图片的内存缓存了，为啥还要加一个NSMapTable来再缓存一次呢？
我想这可能是因为NSCache在收到内存警告时会自动释放缓存，当然这是没有问题的，但坑的是它的释放是没有顺序的，所以可能是刚存入的数据对象被清理了，而不是我们希望的“先进先出”顺序，在实际情况中，往往是最新存入的数据被再次用到的可能性比较大，所以作者在NSCache的基础上又加了一个NSMapTable缓存，这应该是为了提高内存缓存的命中率吧

NSCache的相关内容可以参考这篇文章 http://nshipster.cn/nscache/

我们再来看一下具体的代码实现，作者重写了NSCache的方法来实现了NSMapTable的缓存，为了方便阅读，我删除了线程安全的代码
```
- (void)setObject:(id)obj forKey:(id)key cost:(NSUInteger)g {
    // 先将对象缓存的 NSCache 中
    [super setObject:obj forKey:key cost:g];
    if (key && obj) {
        // 如果存在key和value，则再存到NSMapTable中
        [self.weakCache setObject:obj forKey:key];
    }
}

// 从该方法中，我们可以看到两次的获取缓存，说明NSMapTable确实是用来提高缓存命中率的
- (id)objectForKey:(id)key {
    // 在NSCache中获取缓存对象
    id obj = [super objectForKey:key];
    if (key && !obj) {
        // 如果没有获取的缓存，则再次在NSMapTable中获取缓存
        obj = [self.weakCache objectForKey:key];
        if (obj) {
            // 如果从NSMapTable中获取到了缓存，则再次存入NSCache中
            NSUInteger cost = 0;
            if ([obj isKindOfClass:[UIImage class]]) {
                cost = SDCacheCostForImage(obj);
            }
            [super setObject:obj forKey:key cost:cost];
        }
    }
    return obj;
}
```
+  `SDImageCache` 的磁盘缓存的实现
磁盘缓存是用NSFileManager来实现的，我们直接看缓存函数
```
- (void)_storeImageDataToDisk:(nullable NSData *)imageData forKey:(nullable NSString *)key {
    // 判断将要缓存的路径是否存在，如果不存在，则创建一个
    if (![self.fileManager fileExistsAtPath:_diskCachePath]) {
        [self.fileManager createDirectoryAtPath:_diskCachePath withIntermediateDirectories:YES attributes:nil error:NULL];
    }
    // 获取默认的缓存路径
    NSString *cachePathForKey = [self defaultCachePathForKey:key];
    // 将路径转为url
    NSURL *fileURL = [NSURL fileURLWithPath:cachePathForKey];
    // 将图片数据写入文件，并保存
    [imageData writeToURL:fileURL options:self.config.diskCacheWritingOptions error:nil];
    
    // 禁用icloud备份，默认是YES
    if (self.config.shouldDisableiCloud) {
        [fileURL setResourceValue:@YES forKey:NSURLIsExcludedFromBackupKey error:nil];
    }
}
```
有该函数可以看出，磁盘缓存是将图片保存的沙盒的cache目录下的，那么最关键的图片的保存路径是怎么来的呢？我们可以看下面这个函数
```
// key：这个key就是图片的url
- (nullable NSString *)cachedFileNameForKey:(nullable NSString *)key {
    const char *str = key.UTF8String;
    // 使用了MD5进行加密处理
    unsigned char r[CC_MD5_DIGEST_LENGTH];
    CC_MD5(str, (CC_LONG)strlen(str), r);
    NSURL *keyURL = [NSURL URLWithString:key];
    NSString *ext = keyURL ? keyURL.pathExtension : key.pathExtension;
    NSString *filename = [NSString stringWithFormat:@"%02x%02x%02x%02x%02x%02x%02x%02x%02x%02x%02x%02x%02x%02x%02x%02x%@",
                          r[0], r[1], r[2], r[3], r[4], r[5], r[6], r[7], r[8], r[9], r[10],
                          r[11], r[12], r[13], r[14], r[15], ext.length == 0 ? @"" : [NSString stringWithFormat:@".%@", ext]];
    // 所以最后的图片保存路径就是 "沙盒cache路径"+"url的md5吗"+".图片类型"
    return filename;
}
```
其它关于缓存的函数
移除过期的缓存或当缓存到最大值时移除较早的图片
```
- (void)deleteOldFilesWithCompletionBlock:(nullable SDWebImageNoParamsBlock)completionBlock {
    dispatch_async(self.ioQueue, ^{
        NSURL *diskCacheURL = [NSURL fileURLWithPath:self.diskCachePath isDirectory:YES];
        NSArray<NSString *> *resourceKeys = @[NSURLIsDirectoryKey, NSURLContentModificationDateKey, NSURLTotalFileAllocatedSizeKey];

        // 获取缓存文件的属性
        NSDirectoryEnumerator *fileEnumerator = [self.fileManager enumeratorAtURL:diskCacheURL
                                                       includingPropertiesForKeys:resourceKeys
                                                                          options:NSDirectoryEnumerationSkipsHiddenFiles
                                                                     errorHandler:NULL];
        // 最早的有效缓存的时间，小于这个时间的缓存都失效了
        NSDate *expirationDate = [NSDate dateWithTimeIntervalSinceNow:-self.config.maxCacheAge];
        NSMutableDictionary<NSURL *, NSDictionary<NSString *, id> *> *cacheFiles = [NSMutableDictionary dictionary];
        // 当前所有缓存的大小
        NSUInteger currentCacheSize = 0;
        // 存储需要移除的缓存图片的路径
        NSMutableArray<NSURL *> *urlsToDelete = [[NSMutableArray alloc] init];
        for (NSURL *fileURL in fileEnumerator) {
            NSError *error;
            NSDictionary<NSString *, id> *resourceValues = [fileURL resourceValuesForKeys:resourceKeys error:&error];
            // 错误处理
            if (error || !resourceValues || [resourceValues[NSURLIsDirectoryKey] boolValue]) {
                continue;
            }
            // 通过时间来判断出需要移除的缓存
            NSDate *modificationDate = resourceValues[NSURLContentModificationDateKey];
            if ([[modificationDate laterDate:expirationDate] isEqualToDate:expirationDate]) {
                [urlsToDelete addObject:fileURL];
                continue;
            }
            // 计算有效缓存大小
            NSNumber *totalAllocatedSize = resourceValues[NSURLTotalFileAllocatedSizeKey];
            currentCacheSize += totalAllocatedSize.unsignedIntegerValue;
            cacheFiles[fileURL] = resourceValues;
        }
        // 移除缓存
        for (NSURL *fileURL in urlsToDelete) {
            [self.fileManager removeItemAtURL:fileURL error:nil];
        }

        // 当缓存大小大于设置的最多缓存控件时，移除相对较早缓存
        if (self.config.maxCacheSize > 0 && currentCacheSize > self.config.maxCacheSize) {
            // 只留下剩下最大缓存的一半，其它全部清除了
            const NSUInteger desiredCacheSize = self.config.maxCacheSize / 2;

            // 通过缓存的时间来排序，才好移除早期的缓存
            NSArray<NSURL *> *sortedFiles = [cacheFiles keysSortedByValueWithOptions:NSSortConcurrent
                                                                     usingComparator:^NSComparisonResult(id obj1, id obj2) {
                                                                         return [obj1[NSURLContentModificationDateKey] compare:obj2[NSURLContentModificationDateKey]];
                                                                     }];
            // 开始清除缓存
            for (NSURL *fileURL in sortedFiles) {
                if ([self.fileManager removeItemAtURL:fileURL error:nil]) {
                    NSDictionary<NSString *, id> *resourceValues = cacheFiles[fileURL];
                    NSNumber *totalAllocatedSize = resourceValues[NSURLTotalFileAllocatedSizeKey];
                    currentCacheSize -= totalAllocatedSize.unsignedIntegerValue;
                    // 当剩下的缓存小于最大缓存的一半时，停止缓存清除
                    if (currentCacheSize < desiredCacheSize) {
                        break;
                    }
                }
            }
        }
        if (completionBlock) {
            dispatch_async(dispatch_get_main_queue(), ^{
                completionBlock();
            });
        }
    });
}
```
* 在SDWebImage中我们可以对其缓存方式进行设置，比如不需要内存缓存、缓存最大容量等，SDWebImage 为我们提供了一个专门配置的对象
```
@interface SDImageCacheConfig : NSObject
// 是否对图片进行解压缩 默认 YSE
@property (assign, nonatomic) BOOL shouldDecompressImages;
// 是否禁用icloud备份 默认 YSE
@property (assign, nonatomic) BOOL shouldDisableiCloud;
// 是否内存缓存 默认 YSE
@property (assign, nonatomic) BOOL shouldCacheImagesInMemory;
/**
 * The reading options while reading cache from disk.
 * Defaults NSDataReadingMappedIfSafe
 */
@property (assign, nonatomic) NSDataReadingOptions diskCacheReadingOptions;
/**
 * The writing options while writing cache to disk.
 * Defaults NSDataWritingAtomic
 */
@property (assign, nonatomic) NSDataWritingOptions diskCacheWritingOptions;
// 缓存的超时时间
@property (assign, nonatomic) NSInteger maxCacheAge;
// 最大缓存容量
@property (assign, nonatomic) NSUInteger maxCacheSize;

@end
```
我需要通过哪里来设置这些呢，其实SDImageCache是一个单例，所以只需我们再下载图片之前取到SDImageCache单例，就可以对其参数进行设置，如下
```
// 如果这几行代码写在 AppDelegate 里面，那么就可以对所有的图片下载进行设置
[SDImageCache sharedImageCache].config.maxCacheAge = 60 * 60 * 24 * 7; // 磁盘缓存 7天
[SDImageCache sharedImageCache].config.maxCacheSize = 0; // 磁盘缓存 这里设置为0，表示无限大
[SDImageCache sharedImageCache].config.shouldCacheImagesInMemory = true; // 开启内存缓存
[SDImageCache sharedImageCache].maxMemoryCost = 0; // 内存缓存 最大值
[SDImageCache sharedImageCache].maxMemoryCountLimit = 0; // 内存缓存 最大数量
```
注意，是否需要磁盘缓存，是通过下载时传入的，在这里并不能配置，在下载时我们需要传入SDWebImageOptions这个参数，默认是SDWebImageRetryFailed，只有我们传入SDWebImageCacheMemoryOnly时，才不会进行磁盘缓存，其它枚举都会进行磁盘缓存，例如下面UIImageView的扩展
```
// 这是我们平时使用最多的函数，它不需要传入options  默认就是 SDWebImageRetryFailed
- (void)sd_setImageWithURL:(nullable NSURL *)url placeholderImage:(nullable UIImage *)placeholder {
    [self sd_setImageWithURL:url placeholderImage:placeholder options:0 progress:nil completed:nil];
}

// options: 如果这里传入 SDWebImageCacheMemoryOnly，则不进行磁盘缓存
- (void)sd_setImageWithURL:(nullable NSURL *)url placeholderImage:(nullable UIImage *)placeholder options:(SDWebImageOptions)options {
    [self sd_setImageWithURL:url placeholderImage:placeholder options:options progress:nil completed:nil];
}
```

总结
SDWebImage 使用NSCache+NSMapTable来实现了内存缓存，使用NSFileManager来实现磁盘缓存，有超时和超出容量两种清除缓存的策略

最后再截个图![SDImageCache 方法预览](https://upload-images.jianshu.io/upload_images/1258499-888526cd30cf8946.jpeg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
