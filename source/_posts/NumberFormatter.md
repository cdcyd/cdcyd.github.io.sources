---
title: 数字转为人名币大写(Swift4.0)
date: 2017-12-15 18:00:00
tags: "Foundation"
categories: "Swift"
---

在iOS中，对数字的格式化操作，我第一个想到的就是它NumberFormatter，所以我写了下面这个函数
```
extension String {
    func numberRMM() -> String {
        guard let num = Double(self) else {
            return ""
        }
        let format = NumberFormatter()
        format.locale = Locale(identifier: "zh_CN")
        format.numberStyle = .spellOut
        format.minimumIntegerDigits = 1
        format.minimumFractionDigits = 0
        format.maximumFractionDigits = 2
        return format.string(from: NSNumber(value: num)) ?? ""
    }
}
```

经测试后
```
// 输入
print(1234566.05.numberRMM())
// 打印结果 一百二十三万四千五百六十六点〇五
```

这显然不是我们要的结果，不过我们还是可以利用这个结果，再加上字符串替换，来实现我们的需求
首先全部字符串的替换如下函数
```
let formattedString = text.replacingOccurrences(of: "一", with: "壹")
                          .replacingOccurrences(of: "二", with: "贰")
                          .replacingOccurrences(of: "三", with: "叁")
                          .replacingOccurrences(of: "四", with: "肆")
                          .replacingOccurrences(of: "五", with: "伍")
                          .replacingOccurrences(of: "六", with: "陆")
                          .replacingOccurrences(of: "七", with: "柒")
                          .replacingOccurrences(of: "八", with: "捌")
                          .replacingOccurrences(of: "九", with: "玖")
                          .replacingOccurrences(of: "十", with: "拾")
                          .replacingOccurrences(of: "百", with: "佰")
                          .replacingOccurrences(of: "千", with: "仟")
                          .replacingOccurrences(of: "〇", with: "零")
```

之后我们再对这个字符串进行处理

1.先判断是否是整数，如果是整数，则在后面加上`元整`两个字就是我们要的结果了，代码如下
```
// 整数处理
let texts = formattedString.components(separatedBy: "点")
if sept.count > 0 && isInt {
    return texts[0].appending("元整")
}
```

2.如果是小数，此时无论有多少位小数，我们都需要保留两位小数，即角和分，后面的数字直接丢弃掉(实际业务中也不会出现有2位小数以上的数)，处理方法如下
```
// 小数处理
let decStr = sept[1]
intStr = intStr.appending("元").appending("\(decStr.first!)角")
if decStr.count > 1 {
    intStr = intStr.appending("\(decStr[decStr.index(decStr.startIndex, offsetBy: 1)])分")
} else {
    intStr = intStr.appending("零分")
}
return intStr
```

经过处理后再次测试如下
```
print(666.005.numberRMM()) // 陆佰陆拾陆元整
print(666.00.numberRMM())  // 陆佰陆拾陆元整
print(666.05.numberRMM())  // 陆佰陆拾陆元零角伍分
print(666.10.numberRMM())  // 陆佰陆拾陆元壹角零分
print(666.1.numberRMM())   // 陆佰陆拾陆元壹角零分
print(666.numberRMM())     // 陆佰陆拾陆元整
```

最后再贴一下完整的代码
```
extension Double {
    func numberRMM() -> String {
        return String(self).numberRMM()
    }
}
extension String {
    /// 人名币大写
    func numberRMM() -> String {
        guard let num = Double(self) else {
            return ""
        }
        let format = NumberFormatter()
        format.locale = Locale(identifier: "zh_CN")
        format.numberStyle = .spellOut
        format.minimumIntegerDigits = 1
        format.minimumFractionDigits = 0
        format.maximumFractionDigits = 2
        let text = format.string(from: NSNumber(value: num)) ?? ""
        let sept = self.components(separatedBy: ".")
        let decimals: Double? = sept.count == 2 ? Double("0." + sept.last!) : nil
        return self.formatRMM(text: text, isInt: decimals == nil || decimals! < 0.01)
    }

    private func formatRMM(text: String, isInt: Bool) -> String {
        let formattedString = text.replacingOccurrences(of: "一", with: "壹")
                                  .replacingOccurrences(of: "二", with: "贰")
                                  .replacingOccurrences(of: "三", with: "叁")
                                  .replacingOccurrences(of: "四", with: "肆")
                                  .replacingOccurrences(of: "五", with: "伍")
                                  .replacingOccurrences(of: "六", with: "陆")
                                  .replacingOccurrences(of: "七", with: "柒")
                                  .replacingOccurrences(of: "八", with: "捌")
                                  .replacingOccurrences(of: "九", with: "玖")
                                  .replacingOccurrences(of: "十", with: "拾")
                                  .replacingOccurrences(of: "百", with: "佰")
                                  .replacingOccurrences(of: "千", with: "仟")
                                  .replacingOccurrences(of: "〇", with: "零")
        let sept = formattedString.components(separatedBy: "点")
        var intStr = sept[0]
        if sept.count > 0 && isInt {
            // 整数处理
            return intStr.appending("元整")
        } else {
            // 小数处理
            let decStr = sept[1]
            intStr = intStr.appending("元").appending("\(decStr.first!)角")
            if decStr.count > 1 {
                intStr = intStr.appending("\(decStr[decStr.index(decStr.startIndex, offsetBy: 1)])分")
            } else {
                intStr = intStr.appending("零分")
            }
            return intStr
        }
    }
}
```

