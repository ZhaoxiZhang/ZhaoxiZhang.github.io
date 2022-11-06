---
title: 减小APK大小
date: 2018-04-21
tags: Android——性能优化
toc: true
---
本篇文章翻译自[Reduce APK Size]( https://developer.android.com/topic/performance/reduce-apk-size.html#apk-structure)

  用户通常不会去下载体积过大的应用程序，特别是当自己的设备连接的是 2G/3G 或者按字节付费的网络。这篇文章描述了如何缩减 APK 的体积大小，以使得更多用户愿意下载你开发的应用。
<!--more-->
## 了解APK结构
  在讨论如何缩减你应用的体积之前，了解 APK 结构是非常有益处的。一个 APK 文件包含了一个 ZIP 文件，该 ZIP 文件包含了组成你应用的所有文件，这些文件包括 Java 字节码文件、资源文件和已编译资源的文件。
APK 包含下列目录：
- <code>META-INF/</code>：包含了<code>CERT.SF</code>、<code>CERT.RSA</code>签名文件以及<code>MAINFEST.MF</code>mainfest文件
- <code>assets/</code>：包含了应用程序的资源，应用程序可以通过 [AssetManager](https://developer.android.com/reference/android/content/res/AssetManager.html) 检索资源
- <code>res/</code>：包含了没有编译到<code>resources.arsc</code>的资源
- <code>lib/</code>：包含了特定处理器的软件层的编译代码，该目录包含了每个平台类型的子目录，例如<code>armeabi</code>、<code>armeabi-v7a</code>、<code>arm64-v8a</code>、<code>x86</code>、<code>x86_64</code>和<code>mips</code>

APK 也包含了下列文件，在这些文件之中，只有<code>Mainfest.xml</code>是强制的
- <code>resources.arsc</code>：包含了编译的资源，该文件包含了来自<code>res/values/</code>目录下所有配置的XML内容。打包工具提取此XML内容，将其编译为二进制格式，并归档内容。此内容包含了语言字符串和样式，以及未直接包含在<code>resources.arsc</code>文件中的内容路径，比如布局文件和图像。
<code>classes.dex</code>：包含了能被<code>Dalvik/ART</code>虚拟机识别的 DEX 文件格式编译的类
<code>AndroidManifest.xml</code>：包含了 Android 核心的 mainfest 文件。该文件罗列了应用程序的名字、版本、权限和引用的第三方库。该文件使用 Android 的二进制 XML 格式

## 减少资源数量和大小
  APK 的大小会影响到应用程序启动的速度、使用的内存和消耗的电量。缩减应用程序大小最简单的方式之一就是减少它所包含的资源数量和大小。特别是你可以移除你的应用中不再使用的资源，或者使用可拓展的 [Drawable](https://developer.android.com/reference/android/graphics/drawable/Drawable.html)  对象来替代图像文件。这部分讨论的这些方法以及一些其他的方式可以减少你应用程序中的资源从而在整体上减少 APK 体积的大小。

### 移除无用的资源
Android Studio中的静态代码检查工具——[lint](https://developer.android.com/studio/write/lint.html)可以检测<code>res/</code>目录下没有引用的资源。当 lint 检查工具发现在你项目中可能存在一个没有使用的资源，它将会打印出类似如下的信息：
```gradle
res/layout/preferences.xml: Warning: The resource R.layout.preferences appears
    to be unused [UnusedResources]
```
>注意：lint 检查工具没有扫描<code>assets/</code>目录，assets 资源是通过反射的方式，或者链接到应用程序的库文件来发现引用的。此外，lint 检查工具并不删除这些资源，它只是提醒你它们的存在。

你使用的一些库可能包含了一些无用的资源，如果你在 <code>build.gradle</code>文件中开启了 [shrinkResources](https://developer.android.com/studio/build/shrink-code.html)，那么 Gradle 可以帮你自动移除这些资源。
```groovy
android {
    // Other settings

    buildTypes {
        release {
            minifyEnabled true
            shrinkResources true
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
        }
    }
}
```
要使用 shrinkResources，你必须启用代码缩减。在编译的过程，首先 [ProGuard](https://developer.android.com/studio/build/shrink-code.html) 移除无用的代码但是并不移除无用的资源，之后由 Gradle 移除无用的资源。
更多有关 ProGuard 和其他一些通过 Android Studio帮助你缩减 APK 体积大小的方法，可以查看[压缩代码和资源](https://developer.android.com/studio/build/shrink-code.html)

### 最小化库中资源的使用
当开发一个 Android 应用的时候，通常会使用一些第三方库来提高应用程序的可用性和多功能性。比如，可能使用了 [Android Support Library](https://developer.android.com/topic/libraries/support-library/index.html) 来改善在旧机型上的用户体验，或者使用 [Google Play Services](https://developers.google.com/android/guides/overview) 为应用程序提供自动翻译。
如果一个库是为服务器或者桌面设计的，那么它通常包含了许多你的应用程序用不到的对象和方法，如果这个库所使用的协议允许，那么你可以修改这个库文件。当然，你也可以使用其他一些对于移动端友好的库来为你的应用程序添加特定功能。
>注意：ProGuard 可以清理第三方库中对你应用非必须的代码，但是它不能移除第三方库的大型内部依赖项。

### 只支持特定的分辨率
Android 支持非常大的设备集，拥有着各式各样的分辨率。在 Android4.4（API level 19）或者更高的系统版本，其框架支持许多分辨率：<code>ldpi</code>、<code>mdpi</code>、<code>tvdpi</code>、<code>hdpi</code>、<code>xhdpi</code>、<code>xxhdpi</code>。尽管 Android 支持所有这些分辨率，但是你并不需要适配每一种分辨率。
如果你知道你的用户群中只有一小部分使用具有特定分辨率的设备，考虑你是否需要适配这些分辨率。如果你没有为特定分辨率准备资源文件，那么 Android 将自动缩放最初为其他屏幕分辨率设计的现有资源。
如果你的应用程序只需要缩放的图片，你可以通过在 <code>drawable-nodpi</code>目录中使用图片的单个版本来节省更多的空间。我们建议每个应用程序至少包含一个<code>xxhdpi</code>图片版本。
更多有关屏幕分辨率的信息，可以查看[屏幕尺寸和密度](https://developer.android.com/about/dashboards/index.html#Screens)

### 使用drawable对象
一些图像并不需要一个静态的图像资源，frameworker 可以在运行时动态的绘制出图像。[Drawable](https://developer.android.com/reference/android/graphics/drawable/Drawable.html) 对象占用APK中少量空间。此外，XML形式的 [Drawable](https://developer.android.com/reference/android/graphics/drawable/Drawable.html) 对象可以产生符合<code>Material Design</code>准则的单色图像。

### 重用资源
你可以为图像的不同分辨率添加一个单独的资源，比如同一图像的着色、阴影或者旋转版本。但是我们强烈建议你重用相同的资源集，在运行时根据需要定制它们。
Android 提供了几个实用程序来更改 asset 的颜色，在 Android5.0（API level 21）或以上版本，可以使用<code>android:tint</code>和<code>tintMode</code>属性。对于较低的系统版本，使用 [ColorFilter](https://developer.android.com/reference/android/graphics/ColorFilter.html) 类
你可以忽略等价于其他资源的资源。下列的代码片段提供了一个例子，通过在图像中间旋转180°将向上的标志转换成向下的标志。
```xml
<?xml version="1.0" encoding="utf-8"?>
<rotate xmlns:android="http://schemas.android.com/apk/res/android"
    android:drawable="@drawable/ic_thumb_up"
    android:pivotX="50%"
    android:pivotY="50%"
    android:fromDegrees="180" />
```

### 从代码中呈现
我们还可以通过程序化渲染图像来缩减 APK 的体积大小。程序化渲染可以释放空间，因为你不再在APK中保存图像文件。

### 压缩PNG文件
<code>aapt</code>工具在编译的过程可以无损压缩存放在<code>res/drawable/</code>目录下的资源。例如，<code>aapt</code>工具可以将不需要超过256种颜色的真彩色PNG转换为具有调色板的8位PNG。 这样会产生质量相同的图像，但内存占用空间更小。
记住aapt有以下限制：
- aapt工具不能压缩<code>asset</code>目录下的PNG文件
- aapt工具只能优化使用不多于256位颜色的图像文件
- aapt工具可能会填充已经压缩过的PNG图像，为了防止这种情况，你可以使用 Gradle 中的<code>cruncherEnabled</code>标志来禁用 PNG 文件的这个过程。
```xml
aaptOptions {
    cruncherEnabled = false
}
```

### 压缩PNG和JPEG文件
你可以使用类似 [pngcrush](https://pmt.sourceforge.io/pngcrush/)、[pngquant](https://pngquant.org/) 或者 [zopflipng](https://github.com/google/zopfli) 等工具来无损压缩 PNG 文件。所有这些工具都可以压缩 PNG 同时保持图像质量。
pngcrush 工具特别有效：这个工具通过使用过滤器和参数的各种组合来压缩图像，在 PNG 过滤器和 zlib(Deflate) 参数上迭代。它选择最小压缩输出的配置。
对于 JPEG 图像，你可以使用类似 [packJPG](http://www.elektronik.htw-aalen.de/packjpg/) 和 [guetzli](https://github.com/google/guetzli) 的工具来压缩。

### 使用WebP文件格式
在Android 3.2（API level 13）或更高版本，除了使用 PNG 或 JPEG 格式的图像文件，你还可以使用 [WebP](https://developers.google.com/speed/webp/) 文件格式的图像。WebP 格式提供有损压缩（类似于JPEG）以及透明度（类似于PNG），但是能更好的提供比 JPEG 或 PNG 更好的压缩效果。
你可以使用 Android Studio 转换 BMP、JPG、PNG或者静态GIF的图像为 WebP 格式。获取更多信息，查看[创建WebP图像](https://developer.android.com/studio/write/convert-webp.html)
>注意：Google Play 只接受[启动图标](https://material.io/guidelines/style/icons.html#)为 PNG 格式的 APKs

### 使用矢量图形
你可以使用矢量图形创建独立于分辨率的图标和其他可伸缩图片。 使用这些图形可以大大减少APK的大小。矢量图形在 Android 中表示为 [VectorDrawable](https://developer.android.com/reference/android/graphics/drawable/VectorDrawable.html) 对象。使用 [VectorDrawable](https://developer.android.com/reference/android/graphics/drawable/VectorDrawable.html) 对象，100字节的文件可以生成屏幕大小的清晰图像。
但是系统将会花费大量时间去渲染 [VectorDrawable](https://developer.android.com/reference/android/graphics/drawable/VectorDrawable.html) 对象，对于大的图像需要更长的时间才能出现在屏幕上。因此，在展示小图像的时候再考虑矢量图。
获取更多有关 [VectorDrawable](https://developer.android.com/reference/android/graphics/drawable/VectorDrawable.html) 对象的信息，查看[使用图片](https://developer.android.com/training/material/drawables.html)

### 使用矢量图形制作动画图像
不要使用 [AnimationDrawable](https://developer.android.com/reference/android/graphics/drawable/AnimationDrawable.html) 去创建逐帧动画，因为这样做需要为动画的每个帧添加单独的位图文件，这会增加 APK 的体积大小。
你可以使用 [AnimatedVectorDrawable](https://developer.android.com/reference/android/support/graphics/drawable/AnimatedVectorDrawableCompat.html) 为[矢量图片添加动画](https://developer.android.com/training/material/animations.html#AnimVector)

## 减少Native和Java代码
你可以使用下列几种方式去减少应用程序中的 Java 和 native 代码库。

### 移除不必要的生成代码
确保了解自动生成的代码的足迹。例如，许多协议缓冲工具生成过多的方法和类，这可能使应用程序的大小增加一倍或者两倍。

### 避免枚举
一个枚举可以为你的应用程序的<code>classes.dex</code>文件增加1.0至1.4KB的大小，对于复杂系统或者共享库，这些将快速累积。如果可能的话，考虑使用<code>@IntDef</code>注解和 [ProGuard](https://developer.android.com/studio/build/shrink-code.html) 来除去枚举并将它们转换为整数。这种类型转换保留了枚举的所有类型安全的好处。

### 减小本地二进制文件的大小
如果你的应用程序使用 native code 和 Android NDK，你可以通过优化代码来减少应用程序的体积大小。两种有用的技术是删除调试符号和避免提取本地库。
- 删除调试符号
如果你的应用程序正在开发并且需要调试，那么使用调试符号是很有意义的。使用 Android NDK 提供的<code>arm-eabi-strip</code>工具去移除本地库中不必要的调试符号，之后再进行 release版本的编译。
- 避免提取本地库
将<code>.so</code>文件保存在 APK 中未压缩的文件，设置<code> application </code>中的元素<code>android:extracNativeLibs</code>为false，这将防止 [PackageManager](https://developer.android.com/reference/android/content/pm/PackageManager.html) 在安装过程中将<code>.so</code>文件从APK复制到文件系统，并且有一个额外的好处，使得应用程序的增量更新更小。

## 维护多个精简版APK
你的 APK 可以包含用户下载但从未使用的内容，例如区域或语言信息。为了让用户下载尽量小的应用程序，你可以将你的应用程序根据屏幕尺寸或GPU纹理支持等因素细分为多个 APKs。
当用户下载你的应用程序的时候，根据他们的设备特点以及设置详情，他们会接收到正确的 APK，这样，设备不会接收设备没有的功能的资源。例如，一个用户有<code>hdpi</code>的设备，他们不需要为更高分辨率的设备提供的<code>xxxhdpi</code>的资源。
获取更多相关信息，可以查看 [Configure APK Splits](https://developer.android.com/studio/build/configure-apk-splits.html) 和 [ Maintaining Multiple APKs](https://developer.android.com/google/play/publishing/multiple-apks.html)