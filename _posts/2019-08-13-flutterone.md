---
layout: post
title: "flutter 基础入门"
date: 2019-08-13
description: "flutter 基础入门(1)"
tag: android
---
flutter 基础入门

<h3> 使用镜像</h3>

由于在国内访问Flutter有时可能会受到限制，Flutter官方为中国开发者搭建了临时镜像，大家可以将如下环境变量加入到用户环境变量中：

```java
export PUB_HOSTED_URL=https://pub.flutter-io.cn
export FLUTTER_STORAGE_BASE_URL=https://storage.flutter-io.cn
```

**注意：** 此镜像为临时镜像，并不能保证一直可用，读者可以参考详情请参考 [Using Flutter in China](https://github.com/flutter/flutter/wiki/Using-Flutter-in-China) 以获得有关镜像服务器的最新动态。

<h3>系统要求</h3>

要安装并运行Flutter，您的开发环境必须满足以下最低要求:

- **操作系统**: Windows 7 或更高版本 (64-bit)

- **磁盘空间**: 400 MB (不包括Android Studio的磁盘空间).

- 工具: Flutter 依赖下面这些命令行工具.

  - [Git for Windows](https://git-scm.com/download/win) (Git命令行工具)

    如果已安装Git for Windows，请确保命令提示符或PowerShell中运行 `git` 命令，不然在后面运行`flutter doctor`时将出现`Unable to find git in your PATH`错误, 此时需要手动添加`C:\Program Files\Git\bin`至`Path`系统环境变量中。

<h3>获取Flutter SDK</h3>

1. 去flutter官网下载其最新可用的安装包

2. 将安装包zip解压到你想安装Flutter SDK的路径

3. 在Flutter安装目录的`flutter`文件下找到`flutter_console.bat`，双击运行并启动**flutter命令行**，接下来，你就可以在Flutter命令行运行flutter命令了。

   > **注意：** 由于一些`flutter`命令需要联网获取数据，如果您是在国内访问，由于众所周知的原因，直接访问很可能不会成功。 上面的`PUB_HOSTED_URL`和`FLUTTER_STORAGE_BASE_URL`是google为国内开发者搭建的临时镜像。详情请参考 [Using Flutter in China](https://github.com/flutter/flutter/wiki/Using-Flutter-in-China)

 更新环境变量

要在终端运行 `flutter` 命令， 你需要添加以下环境变量到系统PATH：

- 转到 “控制面板>用户帐户>用户帐户>更改我的环境变量”
- 在“用户变量”下检查是否有名为“Path”的条目:
  - 如果该条目存在, 追加 `flutter\bin`的全路径，使用 `;` 作为分隔符.
  - 如果条目不存在, 创建一个新用户变量 `Path` ，然后将 `flutter\bin`的全路径作为它的值.
- 在“用户变量”下检查是否有名为”PUB_HOSTED_URL”和”FLUTTER_STORAGE_BASE_URL”的条目，如果没有，也添加它们。

重启Windows以应用此更改

<h3> 编辑器设置</h3>

###### 安装Android Studio

- [Android Studio](https://developer.android.com/studio/index.html), 3.0或更高版本.

###### 安装Flutter和Dart插件

需要安装两个插件:

- `Flutter`插件： 支持Flutter开发工作流 (运行、调试、热重载等).
- `Dart`插件： 提供代码分析 (输入代码时进行验证、代码补全等).

<h3> 创建新应用</h3>

1. 选择 **File>New Flutter Project**

2. 选择 **Flutter application** 作为 project 类型, 然后点击 Next

   ![QQ截图20190813151753](/images/posts/flutter/QQ截图20190813151753.png)

3. 输入项目名称 (如 `myapp`), 然后点击 Next

4. 输入自己的Company domain 点击 **Finish**

5. 等待Android Studio安装SDK并创建项目.  

<h3> 运行应用程序</h3>

![Main IntelliJ toolbar](https://flutterchina.club/images/intellij/main-toolbar.png)

在工具栏中点击 **Run图标**, 或者调用菜单项 **Run > Run**.

![16671C9696A0162B58D614A8DE7281E5](/images/posts/flutter/16671C9696A0162B58D614A8DE7281E5.jpg)