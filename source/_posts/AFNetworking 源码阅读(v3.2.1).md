---
title: AFNetworking 源码阅读(v3.2.1)
date: 2018-07-04 17:04:00
tags: "AFNetworking"
categories: "源码阅读"
---

AFNetworking项目地址 https://github.com/AFNetworking/AFNetworking
下载打开后目录
![AFNetworking](https://upload-images.jianshu.io/upload_images/1258499-be018d56578131c7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
1.AFNetworking文件下是实现HTTP请求的类
2.UIKit+AFNetworking文件下是实现图片下载的类

下面我们主要看AFNetworking的HTTP请求实现，我们使用AF发送一个请求很简单，如下面的一个GET请求的例子
```
// 请求管理器
AFHTTPSessionManager *manager = [AFHTTPSessionManager manager];
// 参数JSON格式
manager.requestSerializer = [AFJSONRequestSerializer serializerWithWritingOptions:NSJSONWritingPrettyPrinted];
// 返回JSON格式
manager.responseSerializer = [AFJSONResponseSerializer serializerWithReadingOptions:NSJSONReadingAllowFragments];
// 超时时间
manager.requestSerializer.timeoutInterval = 30;
// 最大线程数
manager.operationQueue.maxConcurrentOperationCount = 5;
// 开始发送请求
[manager GET:@"url" parameters:nil progress:nil success:^(NSURLSessionDataTask * _Nonnull task, id  _Nullable responseObject) {
    // 请求成功
} failure:^(NSURLSessionDataTask * _Nullable task, NSError * _Nonnull error) {
    // 请求失败
}];
```
上面涉及到了`AFHTTPSessionManager`、`AFJSONRequestSerializer`、`AFJSONResponseSerializer`几个类，下面我们就来挨个看他们都是干什么的
* AFJSONRequestSerializer：请求参数按JSON格式序列化，还有一个AFPropertyListRequestSerializer：请求参数按Plist格式序列化，它们都继承自AFHTTPRequestSerializer，而AFHTTPRequestSerializer的参数序列化格式是默认的普通的http的编码格式，即url?key1=value1&key2=value2，下面我们就来看一下AFHTTPRequestSerializer
```
@interface AFHTTPRequestSerializer : NSObject <AFURLRequestSerialization>

/// 返回参数编码的编码样式，默认为 NSUTF8StringEncoding
@property (nonatomic, assign) NSStringEncoding stringEncoding;

/// 是否可以通过手机网络发送请求
@property (nonatomic, assign) BOOL allowsCellularAccess;

/// 缓存策略
@property (nonatomic, assign) NSURLRequestCachePolicy cachePolicy;

/// 是否对cookies进行默认处理  默认为YES
@property (nonatomic, assign) BOOL HTTPShouldHandleCookies;

/// 是否可以在上个数据传输的请求完成后继续传输数据 默认为No
@property (nonatomic, assign) BOOL HTTPShouldUsePipelining;

/// 服务器的类型 默认为NSURLNetworkServiceTypeDefault
@property (nonatomic, assign) NSURLRequestNetworkServiceType networkServiceType;

/// 一个请求的超时时长 默认为60s
@property (nonatomic, assign) NSTimeInterval timeoutInterval;

/// 请求头的信息， 默认包含 Accept-Language 和 User-Agent
@property (readonly, nonatomic, strong) NSDictionary <NSString *, NSString *> *HTTPRequestHeaders;

/// 返回一个默认配置序列化对象
+ (instancetype)serializer;

/// 设置Header里面的字段，如果为field为空，那么这个字段会从Header里面移除
- (void)setValue:(nullable NSString *)value forHTTPHeaderField:(NSString *)field;

/// 根据key取出Hearder里面的值，返回字符串或者nil
- (nullable NSString *)valueForHTTPHeaderField:(NSString *)field;

/// 设置HTTP认证的用户名和密码
- (void)setAuthorizationHeaderFieldWithUsername:(NSString *)username password:(NSString *)password;

/// 移除验证的用户名和密码
- (void)clearAuthorizationHeader;

/// 将参数编码成一个查询字符串（默认包含：GET、DELETE、HEAD）
@property (nonatomic, strong) NSSet <NSString *> *HTTPMethodsEncodingParametersInURI;

/// 根据之前预定义的style设置查询字符串序列化的方法
- (void)setQueryStringSerializationWithStyle:(AFHTTPRequestQueryStringSerializationStyle)style;

/// 根据指定的block设置一个自定义查询字符序列化的方法。Block中传入一个request，编码的参数parameters和一个error，返回请求参数编码成一个查询字符串
- (void)setQueryStringSerializationWithBlock:(nullable NSString * (^)(NSURLRequest *request, id parameters, NSError * __autoreleasing *error))block;

/// 创建一个请求，根据传入的Method，如果为 `GET`、`HEAD`、`DELETE`，参数会拼接在Url的后面，否则参数会设置成HTTP的请求体，并根据request指定的parameterEncoding参数编码
- (NSMutableURLRequest *)requestWithMethod:(NSString *)method
                                 URLString:(NSString *)URLString
                                parameters:(nullable id)parameters
                                     error:(NSError * _Nullable __autoreleasing *)error;

/// 通过`AFMultipartFormData`类型的formData来构建请求体
- (NSMutableURLRequest *)multipartFormRequestWithMethod:(NSString *)method
                                              URLString:(NSString *)URLString
                                             parameters:(nullable NSDictionary <NSString *, id> *)parameters
                              constructingBodyWithBlock:(nullable void (^)(id <AFMultipartFormData> formData))block
                                                  error:(NSError * _Nullable __autoreleasing *)error;

/// 通过request创建一个request，新request的httpBody是`fileURL`指定的文件，并且是通过`HTTPBodyStream`这个属性添加，`HTTPBodyStream`属性的数据会自动添加为httpBody
- (NSMutableURLRequest *)requestWithMultipartFormRequest:(NSURLRequest *)request
                             writingStreamContentsToFile:(NSURL *)fileURL
                                       completionHandler:(nullable void (^)(NSError * _Nullable error))handler;

@end
```
总结一下AFHTTPRequestSerializer，就是封装请求头信息，序列化请求参数
HTTP的头信息包括：请求方式、请求的URL 、HTTP的版本、Host、 Accept、 Cookie、 User-Agent、Accept-Language、Accept-Encoding、Connection等，其中有很多就是AF帮助我们构建的，下面贴出AFHTTPRequestSerializer的初始化方法
```
- (instancetype)init {
    self = [super init];
    if (!self) {
        return nil;
    }

    // 编码方式
    self.stringEncoding = NSUTF8StringEncoding;

    // 存储HTTP头信息
    self.mutableHTTPRequestHeaders = [NSMutableDictionary dictionary];

    // 修改头信息的线程
    self.requestHeaderModificationQueue = dispatch_queue_create("requestHeaderModificationQueue", DISPATCH_QUEUE_CONCURRENT);

    // 构建Accept-Language信息
    NSMutableArray *acceptLanguagesComponents = [NSMutableArray array];
    [[NSLocale preferredLanguages] enumerateObjectsUsingBlock:^(id obj, NSUInteger idx, BOOL *stop) {
        float q = 1.0f - (idx * 0.1f);
        [acceptLanguagesComponents addObject:[NSString stringWithFormat:@"%@;q=%0.1g", obj, q]];
        *stop = q <= 0.5f;
    }];
    [self setValue:[acceptLanguagesComponents componentsJoinedByString:@", "] forHTTPHeaderField:@"Accept-Language"];

    // 构建User-Agent信息（源码这里分类iOS、Watch、MAC三中，这里简化为iOS）
    NSString *userAgent = [NSString stringWithFormat:@"%@/%@ (%@; iOS %@; Scale/%0.2f)", [[NSBundle mainBundle] infoDictionary][(__bridge NSString *)kCFBundleExecutableKey] ?: [[NSBundle mainBundle] infoDictionary][(__bridge NSString *)kCFBundleIdentifierKey], [[NSBundle mainBundle] infoDictionary][@"CFBundleShortVersionString"] ?: [[NSBundle mainBundle] infoDictionary][(__bridge NSString *)kCFBundleVersionKey], [[UIDevice currentDevice] model], [[UIDevice currentDevice] systemVersion], [[UIScreen mainScreen] scale]];
    if (![userAgent canBeConvertedToEncoding:NSASCIIStringEncoding]) {
        NSMutableString *mutableUserAgent = [userAgent mutableCopy];
        if (CFStringTransform((__bridge CFMutableStringRef)(mutableUserAgent), NULL, (__bridge CFStringRef)@"Any-Latin; Latin-ASCII; [:^ASCII:] Remove", false)) {
            userAgent = mutableUserAgent;
        }
    }
    [self setValue:userAgent forHTTPHeaderField:@"User-Agent"];

    // 需要转换为查询字符串参数的请求类型 默认 "GET" "HEAD" "DELETE"
    self.HTTPMethodsEncodingParametersInURI = [NSSet setWithObjects:@"GET", @"HEAD", @"DELETE", nil];

    // 添加一些属性的KVC，包括allowsCellularAccess、cachePolicy、HTTPShouldHandleCookies、HTTPShouldUsePipelining、networkServiceType、timeoutInterval
    self.mutableObservedChangedKeyPaths = [NSMutableSet set];
    for (NSString *keyPath in AFHTTPRequestSerializerObservedKeyPaths()) {
        if ([self respondsToSelector:NSSelectorFromString(keyPath)]) {
            [self addObserver:self forKeyPath:keyPath options:NSKeyValueObservingOptionNew context:AFHTTPRequestSerializerObserverContext];
        }
    }

    return self;
}
```
构建请求对象的方法有3个，如下截图
![构建请求发放申明](https://upload-images.jianshu.io/upload_images/1258499-27975ee4a29a30e0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
下面贴出第一个方法的具体实现
```
- (NSMutableURLRequest *)requestWithMethod:(NSString *)method URLString:(NSString *)URLString parameters:(id)parameters error:(NSError *__autoreleasing *)error
{
    NSParameterAssert(method);
    NSParameterAssert(URLString);

    // 基础url
    NSURL *url = [NSURL URLWithString:URLString];

    NSParameterAssert(url);

    // 请求对象
    NSMutableURLRequest *mutableRequest = [[NSMutableURLRequest alloc] initWithURL:url];
    mutableRequest.HTTPMethod = method;

    // 设置请求参数
    for (NSString *keyPath in AFHTTPRequestSerializerObservedKeyPaths()) {
        if ([self.mutableObservedChangedKeyPaths containsObject:keyPath]) {
            [mutableRequest setValue:[self valueForKeyPath:keyPath] forKey:keyPath];
        }
    }

    // AFURLRequestSerialization协议的方法，用于序列化参数，添加一些header信息
    mutableRequest = [[self requestBySerializingRequest:mutableRequest withParameters:parameters error:error] mutableCopy];

	return mutableRequest;
}

- (NSURLRequest *)requestBySerializingRequest:(NSURLRequest *)request withParameters:(id)parameters error:(NSError *__autoreleasing *)error
{
    NSParameterAssert(request);

    // 深拷贝一份请求对象
    NSMutableURLRequest *mutableRequest = [request mutableCopy];

    // 设置header信息
    [self.HTTPRequestHeaders enumerateKeysAndObjectsUsingBlock:^(id field, id value, BOOL * __unused stop) {
        if (![request valueForHTTPHeaderField:field]) {
            [mutableRequest setValue:value forHTTPHeaderField:field];
        }
    }];

    // 参数序列化
    NSString *query = nil;
    if (parameters) {
        if (self.queryStringSerialization) {
            NSError *serializationError;
            query = self.queryStringSerialization(request, parameters, &serializationError);

            if (serializationError) {
                if (error) {
                    *error = serializationError;
                }

                return nil;
            }
        } else {
            switch (self.queryStringSerializationStyle) {
                case AFHTTPRequestQueryStringDefaultStyle:
                    query = AFQueryStringFromParameters(parameters);
                    break;
            }
        }
    }

    if ([self.HTTPMethodsEncodingParametersInURI containsObject:[[request HTTPMethod] uppercaseString]]) {
        // 将参数转换为查询字符串并拼接到url后面，默认"GET" "HEAD" "DELETE"请求会走这里
        if (query && query.length > 0) {
            mutableRequest.URL = [NSURL URLWithString:[[mutableRequest.URL absoluteString] stringByAppendingFormat:mutableRequest.URL.query ? @"&%@" : @"?%@", query]];
        }
    } else {
        // 其它请求将参数设置成HTTPBody，并且如果没有参数，则设置成""
        if (!query) {
            query = @"";
        }
        if (![mutableRequest valueForHTTPHeaderField:@"Content-Type"]) {
            [mutableRequest setValue:@"application/x-www-form-urlencoded" forHTTPHeaderField:@"Content-Type"];
        }
        [mutableRequest setHTTPBody:[query dataUsingEncoding:self.stringEncoding]];
    }

    return mutableRequest;
}
```
到此，我们就构建NSURLRequest，其实它主要有两部分：构建头信息、处理参数

下面我们再来看一下返回信息AFJSONResponseSerializer
* AFJSONResponseSerializer：它主要用于对返回二进制数据NSData的解析，继承自AFHTTPResponseSerializer，其中的JSON表示它能解析的类型
  1. AFHTTPResponseSerializer：不做处理，直接返回NSData
  2. AFJSONResponseSerializer：JSON
  3. AFXMLParserResponseSerializer：xml
  4. AFXMLDocumentResponseSerializer：(Mac OS X) iPhone不能直接使用，需要用GData-XML
  5. AFPropertyListResponseSerializer：plist
  6. AFImageResponseSerializer：image
  7. AFCompoundResponseSerializer：多个组合，初始化时需要传入上面的一种或几种类型

它们都遵循AFURLResponseSerialization协议，下面贴出AFJSONResponseSerializer的实现方法
```
// AFURLResponseSerialization协议
- (id)responseObjectForResponse:(NSURLResponse *)response data:(NSData *)data error:(NSError *__autoreleasing *)error
{
    // 判断返回的数据是否合法，如果不合法，直接返回nil
    if (![self validateResponse:(NSHTTPURLResponse *)response data:data error:error]) {
        if (!error || AFErrorOrUnderlyingErrorHasCodeInDomain(*error, NSURLErrorCannotDecodeContentData, AFURLResponseSerializationErrorDomain)) {
            return nil;
        }
    }

    // json解析
    BOOL isSpace = [data isEqualToData:[NSData dataWithBytes:" " length:1]];
    if (data.length == 0 || isSpace) {
        return nil;
    }
    NSError *serializationError = nil;
    id responseObject = [NSJSONSerialization JSONObjectWithData:data options:self.readingOptions error:&serializationError];
    if (!responseObject)
    {
        // 解析失败，返回nil
        if (error) {
            *error = AFErrorWithUnderlyingError(serializationError, *error);
        }
        return nil;
    }

    // 是否需要移除nil值的key
    if (self.removesKeysWithNullValues) {
        return AFJSONObjectByRemovingKeysWithNullValues(responseObject, self.readingOptions);
    }
    return responseObject;
}

// 验证数据合法性函数
- (BOOL)validateResponse:(NSHTTPURLResponse *)response data:(NSData *)data error:(NSError * __autoreleasing *)error
{
    // 是否合法
    BOOL responseIsValid = YES;
    // 错误信息
    NSError *validationError = nil;
    // 验证并生成错误信息
    if (response && [response isKindOfClass:[NSHTTPURLResponse class]]) {
        if (self.acceptableContentTypes && ![self.acceptableContentTypes containsObject:[response MIMEType]] && !([response MIMEType] == nil && [data length] == 0)) {
            if ([data length] > 0 && [response URL]) {
                NSString *desc = [NSString stringWithFormat:NSLocalizedStringFromTable(@"Request failed: unacceptable content-type: %@", @"AFNetworking", nil), [response MIMEType]];
                NSMutableDictionary *mutableUserInfo = [@{NSLocalizedDescriptionKey:desc,
                                                          NSURLErrorFailingURLErrorKey:[response URL],
                                                          AFNetworkingOperationFailingURLResponseErrorKey: response} mutableCopy];
                if (data) {
                    mutableUserInfo[AFNetworkingOperationFailingURLResponseDataErrorKey] = data;
                }
                validationError = AFErrorWithUnderlyingError([NSError errorWithDomain:AFURLResponseSerializationErrorDomain code:NSURLErrorCannotDecodeContentData userInfo:mutableUserInfo], validationError);
            }
            responseIsValid = NO;
        }

        if (self.acceptableStatusCodes && ![self.acceptableStatusCodes containsIndex:(NSUInteger)response.statusCode] && [response URL]) {
            NSString *desc = [NSString stringWithFormat:NSLocalizedStringFromTable(@"Request failed: %@ (%ld)", @"AFNetworking", nil), [NSHTTPURLResponse localizedStringForStatusCode:response.statusCode], (long)response.statusCode];
            NSMutableDictionary *mutableUserInfo = [@{NSLocalizedDescriptionKey: desc,
                                                      NSURLErrorFailingURLErrorKey:[response URL],
                                                      AFNetworkingOperationFailingURLResponseErrorKey: response} mutableCopy];
            if (data) {
                mutableUserInfo[AFNetworkingOperationFailingURLResponseDataErrorKey] = data;
            }
            validationError = AFErrorWithUnderlyingError([NSError errorWithDomain:AFURLResponseSerializationErrorDomain code:NSURLErrorBadServerResponse userInfo:mutableUserInfo], validationError);
            responseIsValid = NO;
        }
    }
    if (error && !responseIsValid) {
        *error = validationError;
    }
    return responseIsValid;
}
```
总结AFHTTPResponseSerializer及其子类，是处理NSURLResponse的，主要处理错误码、错误信息、解析返回的NSData等

* AFHTTPSessionManager：请求管理类，它继承自AFURLSessionManager，它主要封装了GET,POST,PUT,DELETE等等HTTPMehtod
```
@interface AFHTTPSessionManager : AFURLSessionManager <NSSecureCoding, NSCopying>

// 基础url
@property (readonly, nonatomic, strong, nullable) NSURL *baseURL;

// 请求x实体
@property (nonatomic, strong) AFHTTPRequestSerializer <AFURLRequestSerialization> * requestSerializer;

// 返回实体
@property (nonatomic, strong) AFHTTPResponseSerializer <AFURLResponseSerialization> * responseSerializer;

// SSL证书验证
@property (nonatomic, strong) AFSecurityPolicy *securityPolicy;

// 实例
+ (instancetype)manager;

// 通过url创建实例
- (instancetype)initWithBaseURL:(nullable NSURL *)url;

// 通过url和config创建实例
- (instancetype)initWithBaseURL:(nullable NSURL *)url
           sessionConfiguration:(nullable NSURLSessionConfiguration *)configuration NS_DESIGNATED_INITIALIZER;

// 发起get请求
- (nullable NSURLSessionDataTask *)GET:(NSString *)URLString
                   parameters:(nullable id)parameters
                      success:(nullable void (^)(NSURLSessionDataTask *task, id _Nullable responseObject))success
                      failure:(nullable void (^)(NSURLSessionDataTask * _Nullable task, NSError *error))failure DEPRECATED_ATTRIBUTE;


// 发起get请求，带进度条
- (nullable NSURLSessionDataTask *)GET:(NSString *)URLString
                            parameters:(nullable id)parameters
                              progress:(nullable void (^)(NSProgress *downloadProgress))downloadProgress
                               success:(nullable void (^)(NSURLSessionDataTask *task, id _Nullable responseObject))success
                               failure:(nullable void (^)(NSURLSessionDataTask * _Nullable task, NSError *error))failure;

// 发起head请求
- (nullable NSURLSessionDataTask *)HEAD:(NSString *)URLString
                    parameters:(nullable id)parameters
                       success:(nullable void (^)(NSURLSessionDataTask *task))success
                       failure:(nullable void (^)(NSURLSessionDataTask * _Nullable task, NSError *error))failure;

// 发起post请求
- (nullable NSURLSessionDataTask *)POST:(NSString *)URLString
                    parameters:(nullable id)parameters
                       success:(nullable void (^)(NSURLSessionDataTask *task, id _Nullable responseObject))success
                       failure:(nullable void (^)(NSURLSessionDataTask * _Nullable task, NSError *error))failure DEPRECATED_ATTRIBUTE;

// 发起post请求，带进度条
- (nullable NSURLSessionDataTask *)POST:(NSString *)URLString
                             parameters:(nullable id)parameters
                               progress:(nullable void (^)(NSProgress *uploadProgress))uploadProgress
                                success:(nullable void (^)(NSURLSessionDataTask *task, id _Nullable responseObject))success
                                failure:(nullable void (^)(NSURLSessionDataTask * _Nullable task, NSError *error))failure;

// 发起post请求，添加formdata，一般用于上传文件
- (nullable NSURLSessionDataTask *)POST:(NSString *)URLString
                    parameters:(nullable id)parameters
     constructingBodyWithBlock:(nullable void (^)(id <AFMultipartFormData> formData))block
                       success:(nullable void (^)(NSURLSessionDataTask *task, id _Nullable responseObject))success
                       failure:(nullable void (^)(NSURLSessionDataTask * _Nullable task, NSError *error))failure DEPRECATED_ATTRIBUTE;

// 发起post请求，添加formdata，一般用于上传文件，带进度条
- (nullable NSURLSessionDataTask *)POST:(NSString *)URLString
                             parameters:(nullable id)parameters
              constructingBodyWithBlock:(nullable void (^)(id <AFMultipartFormData> formData))block
                               progress:(nullable void (^)(NSProgress *uploadProgress))uploadProgress
                                success:(nullable void (^)(NSURLSessionDataTask *task, id _Nullable responseObject))success
                                failure:(nullable void (^)(NSURLSessionDataTask * _Nullable task, NSError *error))failure;

// 发起put请求
- (nullable NSURLSessionDataTask *)PUT:(NSString *)URLString
                   parameters:(nullable id)parameters
                      success:(nullable void (^)(NSURLSessionDataTask *task, id _Nullable responseObject))success
                      failure:(nullable void (^)(NSURLSessionDataTask * _Nullable task, NSError *error))failure;

// 发起patch请求
- (nullable NSURLSessionDataTask *)PATCH:(NSString *)URLString
                     parameters:(nullable id)parameters
                        success:(nullable void (^)(NSURLSessionDataTask *task, id _Nullable responseObject))success
                        failure:(nullable void (^)(NSURLSessionDataTask * _Nullable task, NSError *error))failure;

// 发起delete请求
- (nullable NSURLSessionDataTask *)DELETE:(NSString *)URLString
                      parameters:(nullable id)parameters
                         success:(nullable void (^)(NSURLSessionDataTask *task, id _Nullable responseObject))success
                         failure:(nullable void (^)(NSURLSessionDataTask * _Nullable task, NSError *error))failure;

@end
```
它的实现最后都会汇集到下面两个方法中，一个上传，一个下载
```
- (NSURLSessionDataTask *)POST:(NSString *)URLString
                    parameters:(id)parameters
     constructingBodyWithBlock:(void (^)(id <AFMultipartFormData> formData))block
                      progress:(nullable void (^)(NSProgress * _Nonnull))uploadProgress
                       success:(void (^)(NSURLSessionDataTask *task, id responseObject))success
                       failure:(void (^)(NSURLSessionDataTask *task, NSError *error))failure
{
    // 构建请求实体
    NSError *serializationError = nil;
    NSMutableURLRequest *request = [self.requestSerializer multipartFormRequestWithMethod:@"POST" URLString:[[NSURL URLWithString:URLString relativeToURL:self.baseURL] absoluteString] parameters:parameters constructingBodyWithBlock:block error:&serializationError];
    if (serializationError) {
        if (failure) {
            dispatch_async(self.completionQueue ?: dispatch_get_main_queue(), ^{
                failure(nil, serializationError);
            });
        }

        return nil;
    }

    // 构建上传任务
    __block NSURLSessionDataTask *task = [self uploadTaskWithStreamedRequest:request progress:uploadProgress completionHandler:^(NSURLResponse * __unused response, id responseObject, NSError *error) {
        if (error) {
            if (failure) {
                failure(task, error);
            }
        } else {
            if (success) {
                success(task, responseObject);
            }
        }
    }];

    // 开始任务
    [task resume];

    return task;
}
```
```
- (NSURLSessionDataTask *)dataTaskWithHTTPMethod:(NSString *)method
                                       URLString:(NSString *)URLString
                                      parameters:(id)parameters
                                  uploadProgress:(nullable void (^)(NSProgress *uploadProgress)) uploadProgress
                                downloadProgress:(nullable void (^)(NSProgress *downloadProgress)) downloadProgress
                                         success:(void (^)(NSURLSessionDataTask *, id))success
                                         failure:(void (^)(NSURLSessionDataTask *, NSError *))failure
{
    // 构建请求体
    NSError *serializationError = nil;
    NSMutableURLRequest *request = [self.requestSerializer requestWithMethod:method URLString:[[NSURL URLWithString:URLString relativeToURL:self.baseURL] absoluteString] parameters:parameters error:&serializationError];
    if (serializationError) {
        if (failure) {
            dispatch_async(self.completionQueue ?: dispatch_get_main_queue(), ^{
                failure(nil, serializationError);
            });
        }

        return nil;
    }

    // 构建下载任务
    __block NSURLSessionDataTask *dataTask = nil;
    dataTask = [self dataTaskWithRequest:request
                          uploadProgress:uploadProgress
                        downloadProgress:downloadProgress
                       completionHandler:^(NSURLResponse * __unused response, id responseObject, NSError *error) {
        if (error) {
            if (failure) {
                failure(dataTask, error);
            }
        } else {
            if (success) {
                success(dataTask, responseObject);
            }
        }
    }];

    return dataTask;
}
```
从上面的方法中，我们可以看出都分为3步，构建请求体、构建下载/上传任务、开始任务
构建请求体：它由AFHTTPRequestSerializer完成
构建下载/上传任务：它主要由AFURLSessionManager完成
下面我们就来看一下AFURLSessionManager
* AFURLSessionManager：管理NSURLSession对象，下面是它的头文件注释
```
@interface AFURLSessionManager : NSObject <NSURLSessionDelegate, NSURLSessionTaskDelegate, NSURLSessionDataDelegate, NSURLSessionDownloadDelegate, NSSecureCoding, NSCopying>

// 会话
@property (readonly, nonatomic, strong) NSURLSession *session;

// NSURLSession的队列
@property (readonly, nonatomic, strong) NSOperationQueue *operationQueue;

// 序列化响应数据的对象
@property (nonatomic, strong) id <AFURLResponseSerialization> responseSerializer;

// 安全策略
@property (nonatomic, strong) AFSecurityPolicy *securityPolicy;

#if !TARGET_OS_WATCH
// 网络监控管理者
@property (readwrite, nonatomic, strong) AFNetworkReachabilityManager *reachabilityManager;
#endif

// 当前被管理的任务的集合（包括data upload download）
@property (readonly, nonatomic, strong) NSArray <NSURLSessionTask *> *tasks;

// 当前data的任务集合
@property (readonly, nonatomic, strong) NSArray <NSURLSessionDataTask *> *dataTasks;

// 当前upload的任务集合
@property (readonly, nonatomic, strong) NSArray <NSURLSessionUploadTask *> *uploadTasks;

// 当前download的任务集合
@property (readonly, nonatomic, strong) NSArray <NSURLSessionDownloadTask *> *downloadTasks;

// 回调的队列，如果为空，就在主队列
@property (nonatomic, strong, nullable) dispatch_queue_t completionQueue;

// block会在这个组中调用，如果为空，就使用一个私有的
@property (nonatomic, strong, nullable) dispatch_group_t completionGroup;

// 解决在后台创建上传任务返回nil的bug，默认为NO，如果设为YES，在后台创建上传任务失败会尝试重新创建该任务
@property (nonatomic, assign) BOOL attemptsToRecreateUploadTasksForBackgroundSessions;

// 初始化
- (instancetype)initWithSessionConfiguration:(nullable NSURLSessionConfiguration *)configuration NS_DESIGNATED_INITIALIZER;

// 是否取消未完成的任务来使session失效
// NSURLSession有两个方法：
// -(void)finishTasksAndInvalidate; 标示待完成所有的任务后失效
// -(void)invalidateAndCancel; 标示 立即失效，未完成的任务也将结束
- (void)invalidateSessionCancelingTasks:(BOOL)cancelPendingTasks;

// 下面2个是DataTask相关的方法，对应没有进度条和有进度条
- (NSURLSessionDataTask *)dataTaskWithRequest:(NSURLRequest *)request
                            completionHandler:(nullable void (^)(NSURLResponse *response, id _Nullable responseObject,  NSError * _Nullable error))completionHandler DEPRECATED_ATTRIBUTE;
- (NSURLSessionDataTask *)dataTaskWithRequest:(NSURLRequest *)request
                               uploadProgress:(nullable void (^)(NSProgress *uploadProgress))uploadProgressBlock
                             downloadProgress:(nullable void (^)(NSProgress *downloadProgress))downloadProgressBlock
                            completionHandler:(nullable void (^)(NSURLResponse *response, id _Nullable responseObject,  NSError * _Nullable error))completionHandler;

/// 下面3个是UploadTask相关的方法，对应fileURL/data/request 这三种不同的数据源
- (NSURLSessionUploadTask *)uploadTaskWithRequest:(NSURLRequest *)request
                                         fromFile:(NSURL *)fileURL
                                         progress:(nullable void (^)(NSProgress *uploadProgress))uploadProgressBlock
                                completionHandler:(nullable void (^)(NSURLResponse *response, id _Nullable responseObject, NSError  * _Nullable error))completionHandler;
- (NSURLSessionUploadTask *)uploadTaskWithRequest:(NSURLRequest *)request
                                         fromData:(nullable NSData *)bodyData
                                         progress:(nullable void (^)(NSProgress *uploadProgress))uploadProgressBlock
                                completionHandler:(nullable void (^)(NSURLResponse *response, id _Nullable responseObject, NSError * _Nullable error))completionHandler;
- (NSURLSessionUploadTask *)uploadTaskWithStreamedRequest:(NSURLRequest *)request
                                                 progress:(nullable void (^)(NSProgress *uploadProgress))uploadProgressBlock
                                        completionHandler:(nullable void (^)(NSURLResponse *response, id _Nullable responseObject, NSError * _Nullable error))completionHandler;

// 下面2个是DownloadTask相关的方法
- (NSURLSessionDownloadTask *)downloadTaskWithRequest:(NSURLRequest *)request
                                             progress:(nullable void (^)(NSProgress *downloadProgress))downloadProgressBlock
                                          destination:(nullable NSURL * (^)(NSURL *targetPath, NSURLResponse *response))destination
                                    completionHandler:(nullable void (^)(NSURLResponse *response, NSURL * _Nullable filePath, NSError * _Nullable error))completionHandler;
- (NSURLSessionDownloadTask *)downloadTaskWithResumeData:(NSData *)resumeData
                                                progress:(nullable void (^)(NSProgress *downloadProgress))downloadProgressBlock
                                             destination:(nullable NSURL * (^)(NSURL *targetPath, NSURLResponse *response))destination
                                       completionHandler:(nullable void (^)(NSURLResponse *response, NSURL * _Nullable filePath, NSError * _Nullable error))completionHandler;

// 通过task获取其上传进度
- (nullable NSProgress *)uploadProgressForTask:(NSURLSessionTask *)task;

// 通过ytask获取其下载进度
- (nullable NSProgress *)downloadProgressForTask:(NSURLSessionTask *)task;

// 下面是一些代理和block回调的相关函数
- (void)setSessionDidBecomeInvalidBlock:(nullable void (^)(NSURLSession *session, NSError *error))block;

- (void)setSessionDidReceiveAuthenticationChallengeBlock:(nullable NSURLSessionAuthChallengeDisposition (^)(NSURLSession *session, NSURLAuthenticationChallenge *challenge, NSURLCredential * _Nullable __autoreleasing * _Nullable credential))block;

- (void)setTaskNeedNewBodyStreamBlock:(nullable NSInputStream * (^)(NSURLSession *session, NSURLSessionTask *task))block;

- (void)setTaskWillPerformHTTPRedirectionBlock:(nullable NSURLRequest * _Nullable (^)(NSURLSession *session, NSURLSessionTask *task, NSURLResponse *response, NSURLRequest *request))block;

- (void)setTaskDidReceiveAuthenticationChallengeBlock:(nullable NSURLSessionAuthChallengeDisposition (^)(NSURLSession *session, NSURLSessionTask *task, NSURLAuthenticationChallenge *challenge, NSURLCredential * _Nullable __autoreleasing * _Nullable credential))block;

- (void)setTaskDidSendBodyDataBlock:(nullable void (^)(NSURLSession *session, NSURLSessionTask *task, int64_t bytesSent, int64_t totalBytesSent, int64_t totalBytesExpectedToSend))block;

- (void)setTaskDidCompleteBlock:(nullable void (^)(NSURLSession *session, NSURLSessionTask *task, NSError * _Nullable error))block;

- (void)setDataTaskDidReceiveResponseBlock:(nullable NSURLSessionResponseDisposition (^)(NSURLSession *session, NSURLSessionDataTask *dataTask, NSURLResponse *response))block;

- (void)setDataTaskDidBecomeDownloadTaskBlock:(nullable void (^)(NSURLSession *session, NSURLSessionDataTask *dataTask, NSURLSessionDownloadTask *downloadTask))block;

- (void)setDataTaskDidReceiveDataBlock:(nullable void (^)(NSURLSession *session, NSURLSessionDataTask *dataTask, NSData *data))block;

- (void)setDataTaskWillCacheResponseBlock:(nullable NSCachedURLResponse * (^)(NSURLSession *session, NSURLSessionDataTask *dataTask, NSCachedURLResponse *proposedResponse))block;

- (void)setDidFinishEventsForBackgroundURLSessionBlock:(nullable void (^)(NSURLSession *session))block AF_API_UNAVAILABLE(macos);

- (void)setDownloadTaskDidFinishDownloadingBlock:(nullable NSURL * _Nullable  (^)(NSURLSession *session, NSURLSessionDownloadTask *downloadTask, NSURL *location))block;

- (void)setDownloadTaskDidWriteDataBlock:(nullable void (^)(NSURLSession *session, NSURLSessionDownloadTask *downloadTask, int64_t bytesWritten, int64_t totalBytesWritten, int64_t totalBytesExpectedToWrite))block;

- (void)setDownloadTaskDidResumeBlock:(nullable void (^)(NSURLSession *session, NSURLSessionDownloadTask *downloadTask, int64_t fileOffset, int64_t expectedTotalBytes))block;

@end
```
从上可以知道，任务的创建有3种data(数据)、upload(上传)、download(下载)，它们的实现都很相似，这里从data来分析，下面是构建NSURLSessionDataTask的实现代码
```
- (NSURLSessionDataTask *)dataTaskWithRequest:(NSURLRequest *)request
                               uploadProgress:(nullable void (^)(NSProgress *uploadProgress)) uploadProgressBlock
                             downloadProgress:(nullable void (^)(NSProgress *downloadProgress)) downloadProgressBlock
                            completionHandler:(nullable void (^)(NSURLResponse *response, id _Nullable responseObject,  NSError * _Nullable error))completionHandler {

    __block NSURLSessionDataTask *dataTask = nil;
    url_session_manager_create_task_safely(^{
        // 通过NSURLSession创建NSURLSessionDataTask
        // - (NSURLSessionDataTask *)dataTaskWithRequest:(NSURLRequest *)request;
        // - (NSURLSessionDataTask *)dataTaskWithURL:(NSURL *)url;
        // - (NSURLSessionUploadTask *)uploadTaskWithRequest:(NSURLRequest *)request fromFile:(NSURL *)fileURL;
        // - (NSURLSessionUploadTask *)uploadTaskWithRequest:(NSURLRequest *)request fromData:(NSData *)bodyData;
        // - (NSURLSessionUploadTask *)uploadTaskWithStreamedRequest:(NSURLRequest *)request;
        // - (NSURLSessionDownloadTask *)downloadTaskWithRequest:(NSURLRequest *)request;
        // - (NSURLSessionDownloadTask *)downloadTaskWithURL:(NSURL *)url;
        // - (NSURLSessionDownloadTask *)downloadTaskWithResumeData:(NSData *)resumeData;
        // - (NSURLSessionStreamTask *)streamTaskWithHostName:(NSString *)hostname port:(NSInteger)port
        // - (NSURLSessionStreamTask *)streamTaskWithNetService:(NSNetService *)service
        dataTask = [self.session dataTaskWithRequest:request];
    });

    // 添加AFURLSessionManagerTaskDelegate代理
    [self addDelegateForDataTask:dataTask uploadProgress:uploadProgressBlock downloadProgress:downloadProgressBlock completionHandler:completionHandler];

    return dataTask;
}
```
上面的代码很简单，就是根据方法调用NSURLSession的相关方法创建相关的任务(data、upload、download)等，在添加AFURLSessionManagerTaskDelegate代理
第一步很好理解，就是创建任务，为什么要有第二步呢？为什么还有添加一个代理呢？
首先我们来看一下NSURLSession的代理，它有4个代理，在创建时只要设置一个相当于4个都设置了
```
self.session = [NSURLSession sessionWithConfiguration:self.sessionConfiguration 
                                             delegate:self 
                                             delegateQueue:self.operationQueue];
```
这里设置的代理后，就相对于设置了下面4个代理
```
1.NSURLSessionDelegate
URLSession:didBecomeInvalidWithError:
URLSession:didReceiveChallenge:completionHandler:
URLSessionDidFinishEventsForBackgroundURLSession:

2. NSURLSessionTaskDelegate
URLSession:willPerformHTTPRedirection:newRequest:completionHandler:
URLSession:task:didReceiveChallenge:completionHandler:
URLSession:task:didSendBodyData:totalBytesSent:totalBytesExpectedToSend:
URLSession:task:needNewBodyStream:
URLSession:task:didCompleteWithError:

3. NSURLSessionDataDelegate
URLSession:dataTask:didReceiveResponse:completionHandler:
URLSession:dataTask:didBecomeDownloadTask:
URLSession:dataTask:didReceiveData:
URLSession:dataTask:willCacheResponse:completionHandler:

4. NSURLSessionDownloadDelegate
URLSession:downloadTask:didFinishDownloadingToURL:
URLSession:downloadTask:didWriteData:totalBytesWritten:totalBytesWritten:totalBytesExpectedToWrite:
URLSession:downloadTask:didResumeAtOffset:expectedTotalBytes:
```
这里需要注意的是，这4个代理不一定都会走，它会根据Task的类型走，如DataTask才会走NSURLSessionDataDelegate，这里还有一个问题，就是当有多个任务同时进行时，我们不好区分到底是哪个人物的回调，当然我们可以通过比较dataTask，AFURLSessionManagerTaskDelegate代理就是为了解决这个问题的，我们就用DataTask类型来举例
```
- (void)addDelegateForDataTask:(NSURLSessionDataTask *)dataTask
                uploadProgress:(nullable void (^)(NSProgress *uploadProgress)) uploadProgressBlock
              downloadProgress:(nullable void (^)(NSProgress *downloadProgress)) downloadProgressBlock
             completionHandler:(void (^)(NSURLResponse *response, id responseObject, NSError *error))completionHandler
{
    // 创建代理
    AFURLSessionManagerTaskDelegate *delegate = [[AFURLSessionManagerTaskDelegate alloc] initWithTask:dataTask];
    delegate.manager = self;
    // 设置完成回调block
    delegate.completionHandler = completionHandler;
    // 任务描述
    dataTask.taskDescription = self.taskDescriptionForSessionTasks;
    // 存储task和delegate
    [self setDelegate:delegate forTask:dataTask];
    // 任务上传进度block
    delegate.uploadProgressBlock = uploadProgressBlock;
    // 任务下载进度block
    delegate.downloadProgressBlock = downloadProgressBlock;
}

- (void)setDelegate:(AFURLSessionManagerTaskDelegate *)delegate
            forTask:(NSURLSessionTask *)task
{
    NSParameterAssert(task);
    NSParameterAssert(delegate);
    [self.lock lock];
    self.mutableTaskDelegatesKeyedByTaskIdentifier[@(task.taskIdentifier)] = delegate;
    [self addNotificationObserverForTask:task];
    [self.lock unlock];
}
```
上面代码就是为每个task创建了一个代理，并且将完成、上传进度、下载进度的回调block赋值个代理，再将代理和任务存储到属性mutableTaskDelegatesKeyedByTaskIdentifier中，这样就使得每一个task都有它自己的代理，当task回调时，我们通过它找到delegate，再用delegate调用相关代理方法，然后再在代理方法中回调相关的block，具体实现如下
在NSURLSessionDataDelegate回调时，下面是获取数据完成的代理
```
- (void)URLSession:(NSURLSession *)session task:(NSURLSessionTask *)task didCompleteWithError:(NSError *)error
{
    // 获取任务对应的delegate
    AFURLSessionManagerTaskDelegate *delegate = [self delegateForTask:task];

    // 如果有delegate，则调用相关的代理方法
    if (delegate) {
        [delegate URLSession:session task:task didCompleteWithError:error];
        // 这里任务完成了，移除任务和代理
        [self removeDelegateForTask:task];
    }
    if (self.taskDidComplete) {
        self.taskDidComplete(session, task, error);
    }
}
```
再来看AFURLSessionManagerTaskDelegate代理中相关实现
```
- (void)URLSession:(__unused NSURLSession *)session task:(NSURLSessionTask *)task didCompleteWithError:(NSError *)error
{
    __strong AFURLSessionManager *manager = self.manager;
    __block id responseObject = nil;
    __block NSMutableDictionary *userInfo = [NSMutableDictionary dictionary];
    userInfo[AFNetworkingTaskDidCompleteResponseSerializerKey] = manager.responseSerializer;

    NSData *data = nil;
    if (self.mutableData) {
        data = [self.mutableData copy];
        self.mutableData = nil;
    }

    if (self.downloadFileURL) {
        userInfo[AFNetworkingTaskDidCompleteAssetPathKey] = self.downloadFileURL;
    } else if (data) {
        userInfo[AFNetworkingTaskDidCompleteResponseDataKey] = data;
    }

    if (error) {
        userInfo[AFNetworkingTaskDidCompleteErrorKey] = error;
        dispatch_group_async(manager.completionGroup ?: url_session_manager_completion_group(), manager.completionQueue ?: dispatch_get_main_queue(), ^{
            if (self.completionHandler) {
                self.completionHandler(task.response, responseObject, error);
            }
            dispatch_async(dispatch_get_main_queue(), ^{
                [[NSNotificationCenter defaultCenter] postNotificationName:AFNetworkingTaskDidCompleteNotification object:task userInfo:userInfo];
            });
        });
    } else {
        dispatch_async(url_session_manager_processing_queue(), ^{
            NSError *serializationError = nil;
            responseObject = [manager.responseSerializer responseObjectForResponse:task.response data:data error:&serializationError];
            if (self.downloadFileURL) {
                responseObject = self.downloadFileURL;
            }
            if (responseObject) {
                userInfo[AFNetworkingTaskDidCompleteSerializedResponseKey] = responseObject;
            }
            if (serializationError) {
                userInfo[AFNetworkingTaskDidCompleteErrorKey] = serializationError;
            }
            dispatch_group_async(manager.completionGroup ?: url_session_manager_completion_group(), manager.completionQueue ?: dispatch_get_main_queue(), ^{
                if (self.completionHandler) {
                    self.completionHandler(task.response, responseObject, serializationError);
                }
                dispatch_async(dispatch_get_main_queue(), ^{
                    [[NSNotificationCenter defaultCenter] postNotificationName:AFNetworkingTaskDidCompleteNotification object:task userInfo:userInfo];
                });
            });
        });
    }
}
```
这样就实现每一个任务都有它自己单独的代理，完成后即进行回调，还有一个好处是，当任务完成时，就可以移除代理，这样可以打破block的循环引用，所以我们再AF的block中直接在self不会造成循环引用

下面在看一下SSL相关类AFSecurityPolicy
* AFSecurityPolicy：它是为了验证证书的，至于HTTP和HTTPS的区别，这个在百度上有很多文章，我这里主要看AFSecurityPolicy都有哪些功能
```
@interface AFSecurityPolicy : NSObject <NSSecureCoding, NSCopying>

// 返回SSL Pinning的类型
@property (readonly, nonatomic, assign) AFSSLPinningMode SSLPinningMode;

// 这个属性保存着所有的可用做校验的证书的集合
// AFNetworking默认会搜索工程中所有.cer的证书文件
// 如果想制定某些证书，可使用certificatesInBundle在目标路径下加载证书，然后调用policyWithPinningMode:withPinnedCertificates创建一个本类对象。
// 只要在证书集合中任何一个校验通过，evaluateServerTrust:forDomain: 就会返回true，即通过校验
@property (nonatomic, strong, nullable) NSSet <NSData *> *pinnedCertificates;

// 允许无效或过期的证书，默认是不允许
@property (nonatomic, assign) BOOL allowInvalidCertificates;

// 是否验证证书中的域名domain
@property (nonatomic, assign) BOOL validatesDomainName;

// 返回指定bundle中的证书
+ (NSSet <NSData *> *)certificatesInBundle:(NSBundle *)bundle;

// 默认的实例对象，默认的认证设置为：
// 1. 不允许无效或过期的证书
// 2. 验证domain名称
// 3. 不对证书和公钥进行验证
+ (instancetype)defaultPolicy;

+ (instancetype)policyWithPinningMode:(AFSSLPinningMode)pinningMode;

+ (instancetype)policyWithPinningMode:(AFSSLPinningMode)pinningMode withPinnedCertificates:(NSSet <NSData *> *)pinnedCertificates;

- (BOOL)evaluateServerTrust:(SecTrustRef)serverTrust
                  forDomain:(nullable NSString *)domain;

@end
```

* 还一个问题，就是线程安全问题，在AF中用了很多GCD函数来保证线程安全

下面函数是用来保证任务创建安全的，AF给出的解释是在iOS8.0以前，任务创建有线程安全问题，如果你适配8.0以后的话，就不会用它了
```
static dispatch_queue_t url_session_manager_creation_queue() {
    static dispatch_queue_t af_url_session_manager_creation_queue;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        af_url_session_manager_creation_queue = dispatch_queue_create("com.alamofire.networking.session.manager.creation", DISPATCH_QUEUE_SERIAL);
    });

    return af_url_session_manager_creation_queue;
}

static void url_session_manager_create_task_safely(dispatch_block_t _Nonnull block) {
    if (block != NULL) {
        if (NSFoundationVersionNumber < NSFoundationVersionNumber_With_Fixed_5871104061079552_bug) {
            // Fix of bug
            // Open Radar:http://openradar.appspot.com/radar?id=5871104061079552 (status: Fixed in iOS8)
            // Issue about:https://github.com/AFNetworking/AFNetworking/issues/2093
            dispatch_sync(url_session_manager_creation_queue(), block);
        } else {
            block();
        }
    }
}
```
下面是处理代理回调的线程函数，它是并行队列，在多个回调同时触发时，可以同时处理，可以加快数据的处理速度
```
static dispatch_queue_t url_session_manager_processing_queue() {
    static dispatch_queue_t af_url_session_manager_processing_queue;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        af_url_session_manager_processing_queue = dispatch_queue_create("com.alamofire.networking.session.manager.processing", DISPATCH_QUEUE_CONCURRENT);
    });
    return af_url_session_manager_processing_queue;
}
```
下面是完成时的回调队列，当completionGroup属性为nil时，默认就使用它
```
static dispatch_group_t url_session_manager_completion_group() {
    static dispatch_group_t af_url_session_manager_completion_group;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        af_url_session_manager_completion_group = dispatch_group_create();
    });
    return af_url_session_manager_completion_group;
}
```
下面再贴出完成时回调的代码
```
if (error) {
        userInfo[AFNetworkingTaskDidCompleteErrorKey] = error;
        // 请求出错
        dispatch_group_async(manager.completionGroup ?: url_session_manager_completion_group(), manager.completionQueue ?: dispatch_get_main_queue(), ^{
            if (self.completionHandler) {
                // 在completionQueue存在时，则completionQueue中回调，否则在主队列中回调
                self.completionHandler(task.response, responseObject, error);
            }
            dispatch_async(dispatch_get_main_queue(), ^{
                // 在主线程中发出任务完成通知
                [[NSNotificationCenter defaultCenter] postNotificationName:AFNetworkingTaskDidCompleteNotification object:task userInfo:userInfo];
            });
        });
    } else {
        // 请求成功
        dispatch_async(url_session_manager_processing_queue(), ^{
            // 子线程并行处理回调数据
            NSError *serializationError = nil;
            responseObject = [manager.responseSerializer responseObjectForResponse:task.response data:data error:&serializationError];
            if (self.downloadFileURL) {
                responseObject = self.downloadFileURL;
            }
            if (responseObject) {
                userInfo[AFNetworkingTaskDidCompleteSerializedResponseKey] = responseObject;
            }
            if (serializationError) {
                userInfo[AFNetworkingTaskDidCompleteErrorKey] = serializationError;
            }
            dispatch_group_async(manager.completionGroup ?: url_session_manager_completion_group(), manager.completionQueue ?: dispatch_get_main_queue(), ^{
                // 处理完成后，在completionQueue存在时，则completionQueue中回调，否则在主队列中回调
                if (self.completionHandler) {
                    self.completionHandler(task.response, responseObject, serializationError);
                }
                dispatch_async(dispatch_get_main_queue(), ^{
                    // 在主线程中发出任务完成通知
                    [[NSNotificationCenter defaultCenter] postNotificationName:AFNetworkingTaskDidCompleteNotification object:task userInfo:userInfo];
                });
            });
        });
    }
```
从上面代码可以看出，在默认情况下，不论我们在哪个线程用AF做请求，它的回调永远是在主队列中

到此AFNetworking文件下的类就全部读完了，实际我读AF就是想了解，在请求时，AF到底都为我们做了什么，总结一下：
1. 请求体NSURLRequest的封装，涉及到构建head信息、Request相关参数设置，请求参数序列化等
2. 根据请求类型创建相关的任务NSURLSessionTask，涉及到任务回调，线程安全等
3. 返回数据NSData的解析
4. HTTPS的支持

这里只是简单的总结这4步，但是每一步的实现都不易，如果想深入了解的话还会涉及到更多的知识点，同时也体会到写一个优秀的网络框架实属不易！