---
title: Unity实现脚本打包
date: 2024-04-15 09:20:00
tags: "Unity"
categories: "CShape"
---

1.安卓打包方式
```
using System.Collections.Generic;
using System.Diagnostics;
using System.IO;
using UnityEditor;
using UnityEngine;

public class EditorBuild : Editor
{
    [MenuItem("Build/Build apk")]
    public static void DebugBuildApk()
    {
        // 获取桌面目录
        string desktop = System.Environment.GetFolderPath(System.Environment.SpecialFolder.Desktop);
        // APK输出路径
        string path = Path.Combine(desktop, "Apk/xxx.apk");
        // 打APK包
        EditorUserBuildSettings.buildAppBundle = false;
        // 开始打包
        BuildPipeline.BuildPlayer(EditorBuildSettings.scenes, path, BuildTarget.Android, BuildOptions.None);
    }

    [MenuItem("Build/Build aab")]
    public static void DebugBuildAab()
    {
        // 获取桌面目录
        string desktop = System.Environment.GetFolderPath(System.Environment.SpecialFolder.Desktop);
        // AAB输出路径
        string path = Path.Combine(desktop, "Apk/xxx.aab");
        // 打AAB包
        EditorUserBuildSettings.buildAppBundle = true;
        // 开始打包
        BuildPipeline.BuildPlayer(EditorBuildSettings.scenes, path, BuildTarget.Android, BuildOptions.None);
    }
}
```

2.iOS打包方式，第一种
```
using System.Collections.Generic;
using System.Diagnostics;
using System.IO;
using UnityEditor;
using UnityEngine;

public class EditorBuild : Editor
{
    // 开始导出XCode工程
    [MenuItem("Build/Build XCode")]
    public static void DebugBuildXCode()
    {
        // 打包证书 TeamID
        PlayerSettings.iOS.appleDeveloperTeamID = "xxxxxxxxxx";
        // 是否自动签名
        PlayerSettings.iOS.appleEnableAutomaticSigning = true;
        // DEBUG宏定义
        // PlayerSettings.SetScriptingDefineSymbolsForGroup(BuildTargetGroup.iOS, "DEBUG");
        // EditorUserBuildSettings.development = true;
        // 设置包名
        PlayerSettings.SetApplicationIdentifier(BuildTargetGroup.iOS, "包名");
        // 桌面目录
        string desktop = System.Environment.GetFolderPath(System.Environment.SpecialFolder.Desktop);
        // 输出路径
        string path = Path.Combine(desktop, "XCodeProject");
        // 开始打包
        BuildPipeline.BuildPlayer(EditorBuildSettings.scenes, path, BuildTarget.iOS, BuildOptions.None);
    }

    // 开始导出IPA包
    [MenuItem("Build/Build IPA")]
    public static void DebugShell()
    {
        // 脚本地址，下面会贴出脚本相关代码
        string scriptPath = "xxx.sh";

        // 创建一个新的进程信息
        ProcessStartInfo startInfo = new ProcessStartInfo
        {
            FileName = "sh",                    // 或者使用 sh，取决于你的系统配置
            Arguments = $"\"{scriptPath}\"",    // 确保路径被正确处理
            UseShellExecute = false,            // 不使用操作系统外壳启动进程
            RedirectStandardOutput = true,      // 重定向标准输出，可选
            RedirectStandardError = true,       // 重定向标准错误，可选
            CreateNoWindow = true               // 不创建新窗口
        };

        // 启动进程
        using (Process process = Process.Start(startInfo))
        {
            // 可以读取标准输出和错误
            string output = process.StandardOutput.ReadToEnd();
            string error = process.StandardError.ReadToEnd();

            // 等待脚本执行完成
            process.WaitForExit();

            // 输出结果
            UnityEngine.Debug.Log(output);
            if (!string.IsNullOrEmpty(error))
            {
                UnityEngine.Debug.Log(error);
            }
        }
    }
}
```

xxx.sh 相关类容
```
#!/bin/bash --login

#1.0 桌面目录
P_DESKTOP="$HOME/Desktop"
#1.1 导出XCode项目路径，注：这里需要与上面的输出路径相同
P_IOS_PROJ=$P_DESKTOP/XCodeProject
#1.2 导出Archive文件目录
P_ARCHIVE=$P_DESKTOP/XCodeArchive
#1.3 导出Archive文件名称
F_ARCHIVE=$P_DESKTOP/XCodeArchive/ios.xcarchive
#1.4 导出IPA文件目录
P_IPA=$P_DESKTOP/IPA
#1.5 Plist文件地址，注：这个文件需要先打第一次包，然后把plist拷贝出来，然后将路径填到这里来
F_EXPORT_PLIST="xxxx.plist"
#1.6 工程名
TARGET_NAME="Unity-iPhone"

#日志输出
echo 导出XCode项目路径:$P_IOS_PROJ
echo 导出Archive文件路径:$F_ARCHIVE
echo 导出IPA文件目录:$P_IPA
echo Plist文件地址:$F_EXPORT_PLIST

#2. 删除工程路径
rm -rf $P_IPA
rm -rf $P_ARCHIVE

cd $P_IOS_PROJ

#3. 清空Build
xcodebuild clean -target "${TARGET_NAME}" -configuration 'Release'

#4. 构建Archive
xcodebuild archive -workspace "${TARGET_NAME}.xcworkspace" -scheme "${TARGET_NAME}" -configuration "Release" -archivePath "${F_ARCHIVE}"

#5. 导出IPA到P_IPA目录
xcodebuild -exportArchive -archivePath "${F_ARCHIVE}" -exportPath "${P_IPA}" -exportOptionsPlist "${F_EXPORT_PLIST}"

#6. 修改文件名
mv $P_IPA/test.ipa $P_IPA/xx_v1.0.0_$(date "+%m%d%H%M").ipa
```

在Unity编辑器上方找到Build，先点击Build XCode，完成后，然后再点击Build IPA，最后就会再桌面上输出API文件夹

3.iOS打包方式2，使用一个脚本来完成
```
#!/bin/bash --login

#1.0 桌面目录
P_DESKTOP="$HOME/Desktop"
#1.0 Unity程序路径
P_UNITY=/Applications/Unity/Hub/Editor/2019.2.21f1/Unity.app/Contents/MacOS/Unity
#1.1 项目工程路径
P_UNITY_PROJ=$(dirname $(realpath $0))/RaiderUnity
#1.2 导出XCode项目路径
P_IOS_PROJ=$P_DESKTOP/XCodeProject
#1.3 导出Archive文件目录
P_ARCHIVE=$P_DESKTOP/XCodeArchive
#1.4 导出Archive文件名称
F_ARCHIVE=$P_DESKTOP/XCodeArchive/ios.xcarchive
#1.5 导出IPA文件目录
P_IPA=$P_DESKTOP/IPA
#1.6 Plist文件地址，注：这个文件需要先打第一次包，然后把plist拷贝出来，然后将路径填到这里来
F_EXPORT_PLIST="xxxx.plist"
#1.7 工程名
TARGET_NAME="Unity-iPhone"

echo Unity安装路径:$P_UNITY
echo 项目工程路径:$P_UNITY_PROJ
echo 导出XCode项目路径:$P_IOS_PROJ
echo 导出Archive文件路径:$F_ARCHIVE
echo 导出IPA文件目录:$P_IPA
echo Plist文件地址:$F_EXPORT_PLIST

#2. 删除工程路径
rm -rf $P_IPA
rm -rf $P_ARCHIVE
rm -rf $P_IOS_PROJ

#3. 创建工程目录
mkdir $P_IOS_PROJ

#4. 开始导出工程
$P_UNITY -projectPath ${P_UNITY_PROJ} -executeMethod EditorBuild.DebugBuild project-$1 -quit

cd $P_IOS_PROJ

#5. 清空Build
xcodebuild clean -target "${TARGET_NAME}" -configuration 'Release'

#6. 构建Archive
xcodebuild archive -workspace "${TARGET_NAME}.xcworkspace" -scheme "${TARGET_NAME}" -configuration "Release" -archivePath "${F_ARCHIVE}"

#7. 导出IPA到P_IPA目录
xcodebuild -exportArchive -archivePath "${F_ARCHIVE}" -exportPath "${P_IPA}" -exportOptionsPlist "${F_EXPORT_PLIST}"

#8. 修改文件名
mv $P_IPA/test.ipa $P_IPA/xx_v1.0.0_$(date "+%m%d%H%M").ipa
```

关闭Unity工程，然后运行脚本
如在Mac系统中
```
./xxx.sh
```
