---
title: 自定义Textfield，解决输入限制和键盘弹出问题
date: 2017-09-12 18:00:00
tags: "UITextfield"
categories: "Objective-C"
---

![Demo截屏](http://upload-images.jianshu.io/upload_images/1258499-e4a26fb02a454a4f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

* 项目由来，最近我开发的项目中，存在很多输入框，它们都有输入限制，比如帐号(限制6位)、密码(限制16位)、手机号(限制只输入数字，11位)、身份证号(限制只输入数字和字母，18位)，金额(限制浮点数)、备注(限制200字)等，类似的输入框还有很多，刚开始我使用UITextField，再加上限制用户输入又是很麻烦的事情，所以一遇到有输入框的vc，就会有大量的限制代码，并且很多都是重复的。在这种情况下，我考虑封装一个TextField，用于解决限制用户输入的功能，顺便在把键盘弹出的问题也解决了
* 所以CCTextField的主要功能，**它能一行代码解决输入限制问题，并且内部处理键盘弹出问题**
* 项目地址：https://github.com/cdcyd/CCTextField 有兴趣的最好把Demo下载看看
* CCTextField 用法，CCTextField继承自UITextField，所以它和UITextField的用法一样，我们只需要多设置一个属性
```
typedef NS_ENUM(NSInteger, CCCheckType){
    CCCheckNone,            // 不做校验
    CCCheckAccount,         // 帐号(字母开头，允许字母、数字、下划线，长度在6个以上)
    CCCheckPassword,        // 密码(以字母开头，只能包含字母、数字和下划线，长度在6个以上)
    CCCheckStrongPassword,  // 强密码(必须包含大小写字母和数字的组合，不能使用特殊字符，长度在6个以上)
    CCCheckEmail,           // 邮箱
    CCCheckZipCode,         // 邮编
    CCCheckDomain,          // 域名
    
    CCCheckPhone,           // 手机号
    CCCheckIDCard,          // 身份证(18位)
    CCCheckFloat,           // 浮点数(校验格式: "10"、"10.0")
    CCCheckDate,            // 日期(校验格式: "xxxx-xx-xx"、"xxxx-x-x")
    CCCheckMoney,           // 金额(校验格式: "10000.0"、"10,000.0"、"10000"、"10,000")
    CCCheckTel,             // 座机(校验格式: "xxx-xxxxxxx"、"xxxx-xxxxxxxx"、"xxx-xxxxxxx"、"xxx-xxxxxxxx"、"xxxxxxx"、"xxxxxxxx")
};
@property(nonatomic, assign)CCCheckType check;
```

 在check的setter方法中，还设置了键盘类型、长度限制等，如果对键盘和输入限制与setter方法设置的不符，则可以在设置check属性之后，再设置键盘类型和长度限制，设置长度限制可以通过下面两个属性设置，但一定要在check之后设置，不然可能会有问题
```
@property(nonatomic, assign)NSInteger minLimit;
@property(nonatomic, assign)NSInteger maxLimit;
```

 所以使用方式如下代码
```
CCTextField *textField = [[CCTextField alloc] initWithFrame:CGRectMake(0, 0, 200, 30)];
// 设置输入类型
textField.check = CCCheckPhone;
// 设置文字最小长度
// textField.minLimit = 0;
// 设置文字最大长度
// textField.maxLimit = 16;
```
