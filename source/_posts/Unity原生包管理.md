---
title: Unity原生包管理
date: 2024-04-15 13:40:00
tags: "Unity"
categories: "CShape"
---

1.包管理工具，使用ExternalDependencyManager，git地址:[https://github.com/googlesamples/unity-jar-resolver]

2.安卓包管理
```
// 新建xxx.xml文件，内容如下
<?xml version="1.0" encoding="utf-8"?>
<dependencies>
    <androidPackages>
        <androidPackage spec="com.squareup.okhttp3:okhttp:3.11.0"/>
        <androidPackage spec="com.squareup.okio:okio:1.14.0"/>
    </androidPackages>
</dependencies>
```

3.iOS包管理
```
// 新建xxx.xml文件，内容如下
<dependencies>
  <iosPods>
    <iosPod name="YYModel" version="1.0.4"/>
  </iosPods>
</dependencies>
```

如果iOS访问私有库，那上面的xml可能无法满足配置，则需要在打包时，对Podfile文件进行修改
```
// 这里官方推荐是45
[PostProcessBuild(45)]
public static void OnPostProcessBuild(BuildTarget target, string buildPath)
{
    if (target == BuildTarget.iOS)
    {
        StreamReader stream = File.OpenText(buildPath + "/Podfile");
        string str = stream.ReadToEnd();
        // 替换需要的设置
        string mdf = str.Replace("xxx", "xxxxxx");
        // 新增team id
        mdf += SetTeamId();
        stream.Dispose();
    
        File.WriteAllText(buildPath + "/Podfile", mdf);
    }
}
```

如果pod库中需要设置teamid，则添加如下代码
```
private static string SetTeamId()
{
    return @"
post_install do |installer|
  installer.generated_projects.each do |project|
    project.targets.each do |target|
      target.build_configurations.each do |config|
        config.build_settings[""DEVELOPMENT_TEAM""] = ""xxxxxx""
      end
    end
  end
end";
}
```
