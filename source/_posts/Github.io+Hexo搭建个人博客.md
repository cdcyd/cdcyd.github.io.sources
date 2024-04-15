---
title: Github.io+Hexo搭建个人博客
date: 2024-04-15 10:45:00
tags: "Blog"
categories: "Hexo"
---

1.Hexo 安装

```
// 安装 Node.js
$ brew install node

// 安装 Hexo
$ sudo npm install -g hexo
```

2.创建源文件
```
$ mkdir xxx.github.io
```

3.初始化hexo
```
// 初始化
$ hexo init xxx.github.io
```

5.安装依赖
```
// 进入目录
$ cd xxx.github.io

// 安装依赖
$ sudo npm install --save hexo-deployer-git
```

6.启动本地服务器
```
hexo server
```

7.浏览器访问[localhost:4000]查看是否完成部署

8.上传网页
```
// (1).生成ssh，打开终端输入，然后根据提示，完成生成ssh
$ ssh-keygen

// (2).查看文件
$ cat ~/.ssh/id_rsa.pub

// (3).创建仓库，登录GitHub，建立名为 [github用户名.github.io] 的仓库

// (4).上传ssh，Github->Settings->SSH and GPG keys->New SSH key->填上标题和key->保存

// (5).使用ssh上传方式, Github->Repositories->xxx.github.io->code->ssh->复制地址->粘贴到博客_config.yml文件中deploy下级repo属性中
```

9.更新上传
```
hexo clean // 清除配置
hexo g     // 生成配置
hexo d -s  // 上传更新网页
```

10.访问发现白屏，因为node.js版本不兼容，14+的版本都不行，可以安装较低版本如:[https://registry.npmmirror.com/binary.html?path=node/v12.16.2/]
