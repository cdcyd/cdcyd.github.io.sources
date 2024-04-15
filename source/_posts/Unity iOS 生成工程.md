---
title: Unity iOS 生成工程
date: 2024-04-15 13:17:00
tags: "Unity"
categories: "CShape"
---

1.生成工程后修改相关配置
```
using System.IO;
using System.Xml;
using UnityEditor;
using UnityEditor.Callbacks;
using UnityEditor.iOS.Xcode;
using UnityEngine;

public static class BuildiOS
{
    // 生成XCode工程是会调用该函数
    [PostProcessBuild(100)]
    public static void OnPostprocessBuild(BuildTarget buildTarget, string buildPath)
    {
        if (buildTarget != BuildTarget.iOS)
            return;
            
        var mProjectPath = PBXProject.GetPBXProjectPath(buildPath);
        var mProject = new PBXProject();

        mProject.ReadFromString(File.ReadAllText(mProjectPath));

        var mFrameworkGUID = GetPBXProjectUnityFrameworkGUID(mProject);
        var mTargetGUID = GetPBXProjectTargetGUID(mProject);
        var mTargetName = GetPBXProjectTargetName(mProject);
        
        // 这里开始修改工程
        // 1.添加系统库
        SystemFrameworkAdd(mProject, mFrameworkGUID);
        
        // 2.修改info.plist文件
        PlistModify(Path.Combine(buildPath, "Info.plist"));
        
        // 3.拷贝三方库(如果有)
        FilesAdd(mProject, mTargetGUID, buildPath, mProjectPath);
        
        // 4.修改Capability(如果需要)
        AddCapability(mProject, buildPath, mTargetGUID, mTargetName);
        
        // 5.修改Entitlements(如果需要)
        AddEntitlements(buildPath, mProject, mTargetGUID, mTargetName);
        
        // 6.修改.h、.m等相关代码(如果需要)
        UnityAppControllerCodesAdd(buildPath);
        
        // 7.保存相关修改
        File.WriteAllText(mProjectPath, mProject.WriteToString());
    }
}

#if UNITY_2019_3_OR_NEWER
    private static string GetPBXProjectTargetName(PBXProject project)
    {
        return "Unity-iPhone";
    }

    private static string GetPBXProjectTargetGUID(PBXProject project)
    {
        return project.GetUnityMainTargetGuid();
    }

    private static string GetPBXProjectUnityFrameworkGUID(PBXProject project)
    {
        return project.GetUnityFrameworkTargetGuid();
    }
#else
    private static string GetPBXProjectTargetName(PBXProject project)
    {
        return PBXProject.GetUnityTargetName();
    }

    private static string GetPBXProjectTargetGUID(PBXProject project)
    {
        return project.TargetGuidByName(PBXProject.GetUnityTargetName());
    }

    private static string GetPBXProjectUnityFrameworkGUID(PBXProject project)
    {
        return GetPBXProjectTargetGUID(project);
    }
#endif
```

2.添加系统库
```
private static void SystemFrameworkAdd(PBXProject project, string mFrameworkGUID)
{
    string[] FRAMEWORKS_TO_ADD = {
        "StoreKit.framework",
    };
    foreach (var framework in FRAMEWORKS_TO_ADD)
    {
        project.AddFrameworkToProject(mFrameworkGUID, framework, false);
    }
}
```

3.修改info.plist文件
```
private static void PlistModify(string plistPath)
{
    var plist = new PlistDocument();
    plist.ReadFromFile(plistPath);

    // 权限说明
    plist.root.SetString("NSPhotoLibraryUsageDescription",
                         "App used your photo library to upload avatar.");

    // 是否使用iOS备用刷新频率
    plist.root.SetBoolean("CADisableMinimumFrameDuration", false);

    // ATS
    PlistElementDict dict0 = plist.root.CreateDict("NSAppTransportSecurity");
    dict0.SetBoolean("NSAllowsArbitraryLoads", true);

    plist.WriteToFile(plistPath);
}
```

4.拷贝三方库(如果有)
```
private static void FilesAdd(PBXProject project, string mTargetGUID, string buildPath, string projectPath)
{
    string ResourcePath = GetResourcePath();
    BuildiOSCopyFile copy = new BuildiOSCopyFile();
    copy.CopyAndAddBuildToXcode(project, mTargetGUID, ResourcePath, buildPath, "xxx文件夹名称");
}
```

5.修改Capability(如果需要)
```
private static void AddCapability(PBXProject project, string buildPath, string targetGUID, string targetName)
{
    // Bitcode
    project.SetBuildProperty(targetGUID, "ENABLE_BITCODE", "NO");
    // Swift
    project.SetBuildProperty(targetGUID, "SWIFT_OBJC_INTERFACE_HEADER_NAME", "Briging-Header-Swift.h");
    // Apple Pay
    project.AddCapability(targetGUID, PBXCapabilityType.ApplePay);
}
```

6.修改.h、.m等相关代码(如果需要)
```
private static void UnityAppControllerCodesAdd(string buildPath)
{
    // 比如修改UnityAppController.mm中的代码
    string mUnityAppControllerPath = buildPath + "/Classes/UnityAppController.mm";
    UnityEditor.XCodeEditor.XClass UnityAppController = new UnityEditor.XCodeEditor.XClass(mUnityAppControllerPath);
    UnityAppController.Replace("xxx", "xxx");
}
```

7.拷贝文件夹代码
```
#if UNITY_IOS
using System.IO;
using UnityEditor.iOS.Xcode;

public static class ExtensionName
{
    public const string META = ".meta";
    public const string ARCHIVE = ".a";
    public const string FRAMEWORK = ".framework";
    public const string BUNDLE = ".bundle";
}

public class BuildiOSCopyFile {

    /// <summary>
    /// 添加编译本地文件到Xcode工程
    /// </summary>
    /// <param name="pbxProject"></param>
    /// <param name="targetGuid"></param>
    /// <param name="oldCopyPath">源路径</param>
    /// <param name="buildPath">拷贝路径</param>
    /// <param name="newCopyPath">拷贝路径</param>
    /// <param name="isAddBuild"></param>
    public void CopyAndAddBuildToXcode(PBXProject pbxProject, string targetGuid, string oldCopyPath, string buildPath, string newCopyPath, bool isAddBuild = true)
    {
        string unityDirectoryPath = oldCopyPath;
        string xcodeDirectoryPath = buildPath;
        if(!string.IsNullOrEmpty(newCopyPath)){
            unityDirectoryPath = Path.Combine(unityDirectoryPath, newCopyPath);
            xcodeDirectoryPath = Path.Combine(xcodeDirectoryPath, newCopyPath);
            Delete (xcodeDirectoryPath);
            Directory.CreateDirectory(xcodeDirectoryPath);
        }
        // 添加 ARCHIVE
        foreach (string filePath in Directory.GetFiles(unityDirectoryPath)) {
            string extension = Path.GetExtension(filePath);
            if(extension == ExtensionName.META)
            {
                continue;
            }
            else if(extension == ExtensionName.ARCHIVE)
            {
                pbxProject.AddBuildProperty(targetGuid, "LIBRARY_SEARCH_PATHS", "$(PROJECT_DIR)/" + newCopyPath);
            }
            string fileName = Path.GetFileName(filePath);
            string copyPath = Path.Combine(xcodeDirectoryPath, fileName);
            if(fileName[0] == '.')
            {
                continue;
            }
            File.Delete(copyPath);
            File.Copy(filePath, copyPath);
            if(isAddBuild)
            {
                string relativePath = Path.Combine(newCopyPath, fileName);
                pbxProject.AddFileToBuild(targetGuid, pbxProject.AddFile(relativePath, relativePath, PBXSourceTree.Source));
            }
        }
        // 遍历文件夹
        foreach (string directoryPath in Directory.GetDirectories(unityDirectoryPath))
        {
            string directoryName = Path.GetFileName(directoryPath);
            bool nextNeedToAddBuild = isAddBuild;
            if(directoryName.Contains(ExtensionName.FRAMEWORK) || directoryName.Contains(ExtensionName.BUNDLE) || directoryName == "Unity-iPhone")
            {
                nextNeedToAddBuild = false;
            }
            CopyAndAddBuildToXcode(pbxProject, targetGuid, oldCopyPath, buildPath, Path.Combine(newCopyPath, directoryName), nextNeedToAddBuild);
            if(directoryName.Contains(ExtensionName.FRAMEWORK) || directoryName.Contains(ExtensionName.BUNDLE))
            {
                string relativePath = Path.Combine(newCopyPath, directoryName);
                string fileGuid = pbxProject.AddFile(relativePath, relativePath, PBXSourceTree.Source);

                pbxProject.AddFileToBuild(targetGuid, fileGuid);
                pbxProject.AddBuildProperty(targetGuid, "FRAMEWORK_SEARCH_PATHS", "$(PROJECT_DIR)/" + newCopyPath);
            }
        }
    }

    /// 拷贝目录（拷贝时候会删除copyPath）
    public void CopyAndReplace(string sourcePath, string copyPath)
    {
        Delete (copyPath);
        Directory.CreateDirectory(copyPath);
        foreach (var file in Directory.GetFiles(sourcePath)){
            File.Copy(file, Path.Combine(copyPath, Path.GetFileName(file)));
        }
        foreach (var dir in Directory.GetDirectories(sourcePath)){
            CopyAndReplace(dir, Path.Combine(copyPath, Path.GetFileName(dir)));
        }
    }

    /// 删除目录
    public void Delete(string targetDirectoryPath){
        if (!Directory.Exists (targetDirectoryPath)) {
            return;
        }
        string[] filePaths = Directory.GetFiles(targetDirectoryPath);
        foreach (string filePath in filePaths){
            File.SetAttributes(filePath, FileAttributes.Normal);
            File.Delete(filePath);
        }
        string[] directoryPaths = Directory.GetDirectories(targetDirectoryPath);
        foreach (string directoryPath in directoryPaths){
            Delete(directoryPath);
        }
        Directory.Delete(targetDirectoryPath, false);
    }
}
#endif
```
