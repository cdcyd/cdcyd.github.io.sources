---
title: CoreLocation框架的定位和逆地址解析详解
date: 2017-11-10 18:00:00
tags: "CoreLocation"
categories: "Swift"
---

### 一、权限问题
在iOS8以后，应用定位需要获取用户授权，我们可以请求的定位权限有两种：
1.仅在使用时定位`requestWhenInUseAuthorization`(应用在前台才能定位)；
2.始终可以定位`requestAlwaysAuthorization`(应用在前后台都可以定位)
在获取权限之前，我们需要在plist文件中添加对应的key，如下图
![Info.plist](http://upload-images.jianshu.io/upload_images/1258499-73ffc9fa2a57b318.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
注意，key后面的value，会在向用户请求权限的弹框中显示，并且会在应用设置->定位中显示，如下图，注意看图中`始终定位`四个字的显示地方
![请求权限弹框](http://upload-images.jianshu.io/upload_images/1258499-8a5e7064deac3b15.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![应用位置设置界面](http://upload-images.jianshu.io/upload_images/1258499-addfd3af7a88a951.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在向plist文件中添加关于定位的key时，一共有四个，如下
1.Privacy - Location When In Use Usage Description
2.Privacy - Location Always and When In Use Usage Description
3.Privacy - Location Always Usage Description
4.Privacy - Location Usage Description
其中4是iOS8之前用的，现在用不到了，所以在添加key时，一定要注意，不要添加错了

还需要注意的是：
1.当只添加Location When In Use Usage Description时，我们只能使用`requestWhenInUseAuthorization`方法请求前台定位的权限
2.当只添加Location Always and When In Use Usage Description时，无论用那个方法请求权限都会报错
This app has attempted to access privacy-sensitive data without a usage description. The app's Info.plist must contain both NSLocationAlwaysAndWhenInUseUsageDescription and NSLocationWhenInUseUsageDescription keys with string values explaining to the user how the app uses this data
我把上面的4个key都测试了遍，发现只有两种添加方式有效：一种是只添加Location When In Use Usage Description，另一种是Location When In Use Usage Description和Location Always and When In Use Usage Description都添加，其他情况都不行(我是iOS11测试的，之前什么情况不记得了)

我们还可以通过代理来获取当前的定位权限，如下：
```
func locationManager(_ manager: CLLocationManager, didChangeAuthorization status: CLAuthorizationStatus) {
    switch status {
    case .notDetermined: print("CoreLocation：用户还未决定授权"); break
    case .restricted: print("CoreLocation：访问受限"); break
    case .denied: print("CoreLocation：用户已授权"); break
    case .authorizedAlways: print("CoreLocation：获得前后台授权"); break
    case .authorizedWhenInUse: print("CoreLocation：获得前台授权"); break
    }
}
```

### 二、获取位置
1.创建定位管理器
```
private let locationManager:CLLocationManager = CLLocationManager()
```
2.配置定位管理器
```
private func setupManager() {
    // kCLLocationAccuracyNearestTenMeters:精度10米  
    // kCLLocationAccuracyHundredMeters:精度100米  
    // kCLLocationAccuracyKilometer:精度1000米  
    // kCLLocationAccuracyThreeKilometers:精度3000米  
    // kCLLocationAccuracyBest:设备使用电池供电时候最高的精度  
    // kCLLocationAccuracyBestForNavigation:导航情况下最高精度
    // 设置定位精度(精度越高越耗电)
    self.locationManager.desiredAccuracy = kCLLocationAccuracyBest
    // 设置定位距离过滤参数，单位是米(当本次定位和上次定位之间的距离大于或等于这个值时，才会调用代理方法) 
    // 如果设为kCLDistanceFilterNone，则每秒更新一次
    self.locationManager.distanceFilter = 10
    // 请求定位权限(注意这个方法只有iOS8以上才有，8之前是不用请求权限的)
    self.locationManager.requestAlwaysAuthorization() 
}
```
3.开始定位
```
func startUpdatingLocation() {
    // 设置代理
    self.locationManager.delegate = self
    // 开始定位(没有网络，也可以定位)
    self.locationManager.startUpdatingLocation()
    // 设置定位30s超时
    self.perform(#selector(LocationManager.locationTimeout), with: nil, afterDelay: 30)
}
```
4.通过定位回调获取位置
```
// 注意，通过该方法获取的坐标是地球坐标(WGS-84)，或者叫GPS坐标
func locationManager(_ manager: CLLocationManager, didUpdateLocations locations: [CLLocation]) {
    // 获取最新位置的坐标
    guard let last = locations.last else {
        return
    }
    print("当前坐标:" + "\(last)")
        
    // 获取到位置后，取消30s的定位超时调用
    NSObject.cancelPreviousPerformRequests(withTarget: self, selector: #selector(LocationManager.locationTimeout), object: nil)
}
```
5.停止定位
```
func stopUpdatingLocation(){
    // 置空代理
    self.locationManager.delegate = nil
    // 停止定位
    self.locationManager.stopUpdatingLocation()
}
```

### 三、CLLocation对象详解
```
    open var coordinate: 经纬度

    open var altitude: 海拔，高度

    open var horizontalAccuracy: 水平的精确度（负数无效）

    open var verticalAccuracy: 垂直的精确度（负数无效）

    open var course: 方向（取值范围是0.0°~359.9°，0.0°代表真北方向）

    open var speed: 当前速度（单位是m/s）

    open var timestamp: 获取位置的时间

    open var floor: 显示楼层的信息，如果当地支持的话，该属性iOS8以后才有

    // 计算两个点之间的距离
    open func distance(from location: CLLocation) -> CLLocationDistance
```

### 四、逆地址解析
我们使用CLGeocoder实现逆地址解析，而且非常简单，如下
```
func reverseGeocodeLocation(location:CLLocation){
    let geocoder = CLGeocoder()
    geocoder.reverseGeocodeLocation(location) { (marks, error) in
        if marks?.first != nil {
            print("当前位置:" + "\(marks!.first!)")
        }
    }
}
```
虽然逆地址解析看似简单，但其中还有很多深坑要填，其主要问题就是坐标系问题，地图坐标系的介绍可以看 [地图坐标系介绍](http://www.cnblogs.com/milkmap/p/3768379.html)

对于`reverseGeocodeLocation`方法，在iOS9中，必须传入地球坐标系(GPS)，而在其他iOS系统中，必须传入火星坐标系(GCJ)

经过测试，我们直接从`didUpdateLocations`方法中获取位置，然后逆地址解析，此时所有iOS系统都不会有问题，但如果我们自己创建一个CLLocation就会有问题，举例子来说明：
```
func reverseGeocodeLocation(location:CLLocation){
    假设lacation参数是通过didUpdateLocations获取后立即传入进来的

    case1:传过来后立即传入 reverseGeocodeLocation逆解析，此时所有系统都不会有问题，如下代码
    let geocoder = CLGeocoder()
    geocoder.reverseGeocodeLocation(location) { (marks, error) in
        if marks?.first != nil {
            print("当前位置:" + "\(marks!.first!)")
        }
    }

    case2:如果此时我从新创建一个CLLocation，此时在iOS9上是没有问题的，但在其他iOS系统上，解析出来就会有很多误差，如下代码
    let loc = CLLocation(latitude: location.coordinate.latitude, longitude: location.coordinate.longitude)
    geocoder.reverseGeocodeLocation(loc) { (marks, error) in
        if marks?.first != nil {
            print("当前位置:" + "\(marks!.first!)")
        }
    }
}
```
所以我猜测CoreLocation内部是有优化的，它用自己的就没有问题，但如果我们自己的坐标，如从后台获取的坐标，此时在逆地址解析的时候就要特别注意了——即9的时候要传GPS坐标，其他时候传火星坐标，这里有一个坐标转换的算法，个人测试了一下还是比较准的 [坐标系转换算法](http://dijkst.github.io/blog/2013/08/09/zhong-guo-di-tu-zuo-biao-pian-yi-suan-fa-zheng-li/)

### 五、CLPlacemark详解
```
    open var location: 地标的坐标

    open var region: 地理区域

    @available(iOS 9.0, *)
    open var timeZone: 时区

    open var name:  名称

    open var thoroughfare: 街道+门牌号

    open var subThoroughfare: 附/子门牌号

    open var locality: 市(如果是直辖市 它总是为nil)

    open var subLocality: 大区

    open var administrativeArea: 省

    open var subAdministrativeArea: 县

    open var postalCode: 邮编

    open var isoCountryCode: 国家代码

    open var country: 国

    open var inlandWater: 内河/湖

    open var ocean: 海洋

    open var areasOfInterest: 关联地标
```
### 六、Demo地址
[定位Demo](https://github.com/cdcyd/CCLocation)
