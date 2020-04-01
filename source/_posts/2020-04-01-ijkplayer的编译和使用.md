---
title: ijkplayer的编译和使用
date: 2020-04-01 09:13:31
tags: 实用
categories: 音视频
---

# 前言

ijkplayer是一个基于FFmpeg的轻量级Android/iOS视频播放器。是一个很优秀的库,但是如果要使用它并不是那么的简单。首先要对ijkplayer进行编译后才能使用。因此下面将分享自己从编译到使用的整个过程，如果有错误欢迎在评论区指出！

# 一、下载并配置Ubuntu虚拟机

1. 根据下面的安装教程，安装Ubuntu虚拟机

   > 温馨提示：在安装过程中为虚拟机分配磁盘大小时，最好选择40GB,自己刚开始选择了20GB，后来需要对内存进行扩容

   [虚拟机安装Ubuntu 16.04.5 图解](https://blog.csdn.net/qq1326702940/article/details/82322079)

2. 如果幸运的话,按照教程就能打开虚拟机,但是我在最后却一直显示黑屏

   解决方法：搜索cmd命令，然后**以管理员方式打开**，输入netsh winsock reset(这个命令是重置网络规范，黑屏的原因很可能就是VMware软件跟本地网络规范有冲突)，回车之后提示成功重置winsock目录，您必须重新启动计算机才能重新完成配置，重启后打开即可。

3. 打开后发现屏幕不能全屏，解决方法就是**安装VMware tools**

   ![**](半屏.png)

4. 安装VMware tools，实现全屏。

   > 当安装VMware tools后还可以进行Windows和Ubuntu进行文本的复制粘贴，这样对后续操作还是挺方便的

   这里参考了网上的解决方法[Ubuntu 16.04下安装VMware Tools（三行命令搞定）](https://blog.csdn.net/luckypython/article/details/77917898?depth_1-utm_source=distribute.pc_relevant.none-task&utm_source=distribute.pc_relevant.none-task)。具体做法：就是打开终端（在桌面鼠标右键就可以看到），然后依次输入下面的命令：

   > 第一步的操作可能需要很久

   - sudo apt-get upgrade
   - sudo apt-get install open-vm-tools-desktop -y
   - sudo reboot

   输入后点击虚拟机的选项，如果变成了**重新安装**就表示安装成功了，如果不行的话就参考这个教程：[Ubuntu下安装VMware Tools的详细过程](https://blog.csdn.net/yibinqi6303/article/details/78382996)

   ![](VMware tools.png)

成功后点击查看，然后点击适应客户机或者全屏就可以实现全屏了

![image-20200331115855435](全屏.png)

# 二、在Ubuntu上安装必要的东西

**1.打开终端，在终端下载git和yasm**

```java
sudo apt-get install git
sudo apt-get install yasm
```

 **2.配置 SDK + NDK**

1. 下载

   - SDK：直接在android studio中文社区下载的，[SDK下载传送门](http://sdk.android-studio.org/)
   - NDK：NDK我试过下载了r21的，结果配置不成功，最后下载了**android-ndk-r10e**这一个版本。[android-ndk-r10e传送门](http://dl.google.com/android/ndk/android-ndk-r10e-linux-x86_64.bin)

2. 解压

   > 当下载之后，两个文件都位于Downloads文件夹下，我把他们都解压到了Home的新建文件夹setup中。

   - SDK：可以直接提取，因为是zip格式

   - NDK：由于是bin模式，所以需要首先对android-ndk-r10e-linux-x86_64.bin文件进行权限设置。设置权限设置可以直接对这个文件进行：右键->属性->允许作为程序可执行文件（打勾）。如下图所示：

     ![image-20200323170925937](NDK解压.png)

   接着执行终端命令：

   ```java
   ./android-ndk-r10e-linux-x86_64.bin 
   ```

3. 配置

   首先得打开配置文件

   ```java
   sudo gedit /etc/profile
   ```

   然后在profile文件的末尾加上下面的路径配置

   > jay是用户名，因为我这些解压文件都放到了Home的setup中，setup是我自己新建的文件夹

   ```java
   export ANDROID_NDK="/home/jay/setup/android-ndk-r10e/" 
   export ANDROID_SDK="/home/jay/setup/android-sdk-linux/" 
   export PATH=$PATH:$ANDROID_NDK
   ```

   保存后使用 `source /etc/profile`使其生效。然后可以用`ndk-build -v`来检验NDK是否配置成功，出现下图则说明配置成功。

   ![image-20200331130855463](F:\md\实用\images\NDK配置成功.png)

   接着开始进行编译

# 三、编译

1. 首先从github中下载ijkplayer，并拉取到ijkplayer-android的文件中。

```
git clone https://github.com/Bilibili/ijkplayer.git ijkplayer-android
```

2. 首先选择ijkplayer-android文件在终端打开，然后进行在终端中输入./init-android.sh初始化，包括了把ffmpeg的代码拉取到本地等操作。

> 友情提示：初始化的过程需要很久的时间

![image-20200323164116451](初始化.png)

```java
./init-android.sh
```

​	我在初始化中途中遇到了如下图的问题：

![image-20200323164514061](F:\md\实用\images\初始化bug.png)

​	原因其实就是git默认的缓存不足，因此需要通过下面的命令来扩大缓存

```
git config --global http.postBuffer 200000000
```

3. 如果要支持HTTPS，还得进行HTTPS的初始化

```java
./init-android-openssl.sh
```

4. 清除一波

```java
cd android/contrib
./compile-openssl.sh clean
./compile-ffmpeg.sh clean
```

> 友情提示：下面的编译过程都很久，这时候可以去干点别的事

5. 编译openssl

```java
./compile-openssl.sh all
```

6. 编译ffmpeg

```java
./compile-ffmpeg.sh all
```

7. 编译ijkplayer,这个编译需要回到上一级android文件上进行编译

```java
cd android
./compile-ijk.sh all
```

​	经过一段时间等待后，当编译成功后就会在ijkplayer-android中/android/ijkplayer的目录，如下：

![image-20200323201743990](ijkplayer目录.png)

# 四、移植到Windows中

1. 确定你的虚拟机是否开启了文件共享，设置路径：虚拟机->设置->选项->共享文件夹，然后选择总是启用的按钮，最后需要重启虚拟机。

![image-20200323205039769](文件共享.png)

2. 接着对编译好的文件进行压缩，不然在复制文件的时候会弹出错误的弹窗。我是对整个ijkplayer-android进行了zip压缩。

3. 接着就是复制粘贴，可以选择拖拽或者是ctrl+c,ctrl+v。
4. 解压，得到ijkplayer-android文件

# 五、播放ijkplayer的sample

1. 在Android Studio中打开ijkplayer-android\android\ijkplayer项目

2. 首先更改项目中的build.gradle，可以根据自己AS的版本来进行更改

   ```java
   // Top-level build file where you can add configuration options common to all sub-projects/modules.
   
   buildscript {
       repositories {
           google()
           jcenter()
       }
       dependencies {
           //根据自己的具体情况更改
           classpath 'com.android.tools.build:gradle:3.5.0'
   
           classpath 'com.github.dcendents:android-maven-gradle-plugin:1.4.1'
           classpath 'com.jfrog.bintray.gradle:gradle-bintray-plugin:1.7'
           // NOTE: Do not place your application dependencies here; they belong
           // in the individual module build.gradle files
       }
   
   
   }
   
   allprojects {
       repositories {
           google()
           jcenter()
       }
   
   }
   
   ext {
       compileSdkVersion = 28        //根据自己的具体情况更改
       buildToolsVersion = "28.0.3"  //根据自己的具体情况更改
   
       targetSdkVersion = 28         //根据自己的具体情况更改
   
       versionCode = 800800
       versionName = "0.8.8"
   }
   
   wrapper {
       gradleVersion = '3.5.0'    //根据自己的具体情况更改
   }
   ```

   

3. 然后更改ijkplayer-example中的build.gradle，主要讲complie修改成implementation，并且修改了下面重点关注注释的地方。这里需要重点关注下添加依赖的地方，如果是在电脑的模拟器运行的话需要ijkplayer-x86，如果是真机运行的话，就需要添加ijkplayer-armv7a。

   ```java
   apply plugin: 'com.android.application'
   
   android {
       .......
        defaultConfig {
           applicationId "tv.danmaku.ijk.media.example"
           minSdkVersion 9
           targetSdkVersion rootProject.ext.targetSdkVersion
           versionCode rootProject.ext.versionCode
           versionName rootProject.ext.versionName
           //重点关注
           flavorDimensions "default"
       }
       ......
       productFlavors {
           //重点关注
           all32 { minSdkVersion 21 }
           all64 { minSdkVersion 21 }
           // armv5 {}
           // armv7a {}
           // arm64 { minSdkVersion 21 }
           // x86 {}
       }
   
       splits {
           abi {
               enable true
               reset()
               include 'x86', 'x86_64', 'armeabi-v7a', 'armeabi'
               universalApk false
           }
       }
   
   }
   
   dependencies {
       implementation fileTree(include: ['*.jar'], dir: 'libs')
       implementation 'com.android.support:appcompat-v7:23.0.1'
       implementation 'com.android.support:preference-v7:23.0.1'
       implementation 'com.android.support:support-annotations:23.0.1'
   
       implementation 'com.squareup:otto:1.3.8'
   
       implementation project(':ijkplayer-java')
       implementation project(':ijkplayer-exo')
   
       //重点关注    
           
       //虚拟机
       api project(':ijkplayer-x86')
       //真机
       api project(':ijkplayer-armv7a')
   
       // compile 'tv.danmaku.ijk.media:ijkplayer-java:0.8.8'
       // compile 'tv.danmaku.ijk.media:ijkplayer-exo:0.8.8'
   
       // all32Compile 'tv.danmaku.ijk.media:ijkplayer-armv5:0.8.8'
       // all32Compile 'tv.danmaku.ijk.media:ijkplayer-armv7a:0.8.8'
       // all32Compile 'tv.danmaku.ijk.media:ijkplayer-x86:0.8.8'
   
       // all64Compile 'tv.danmaku.ijk.media:ijkplayer-armv5:0.8.8'
       // all64Compile 'tv.danmaku.ijk.media:ijkplayer-armv7a:0.8.8'
       // all64Compile 'tv.danmaku.ijk.media:ijkplayer-arm64:0.8.8'
       // all64Compile 'tv.danmaku.ijk.media:ijkplayer-x86:0.8.8'
       // all64Compile 'tv.danmaku.ijk.media:ijkplayer-x86_64:0.8.8'
   
       // armv5Compile project(':player-armv5')
       // armv7aCompile project(':player-armv7a')
       // arm64Compile project(':player-arm64')
       // x86Compile project(':player-x86')
       // x86_64Compile project(':player-x86_64')
   }
   
   ```

4. 这时候你可以试着运行下，根据AS的报错信息进行修改，修改的地方其实就是每个模块下的build.gradle，需要将complie修改成implementation，下面只举一个ijkplayer-java的build.gradle

   ```java
   apply plugin: 'com.android.library'
   
   .......
   dependencies {
       //只需修改这个地方
       implementation fileTree(dir: 'libs', include: ['*.jar'])
   }
   
   apply from: new File(rootProject.projectDir, "tools/gradle-on-demand.gradle")
   ```

5. 编译成功后，运行例子，例子如下（模拟器）：

   <img src="例子1.png" style="zoom:67%;" />

6. 点击Sample，然后点击其中一个例子

   <img src="例子2.png" style="zoom:67%;" />

   <img src="例子3.png" style="zoom:67%;" />

# 六、项目中使用ijkplayer

直接按照下面的文章步骤走，这里我就不再多说了

[Ijkplayer demo 基本使用](https://www.jianshu.com/p/5d1d46aa721d)

 # 总结

整个过程下来花费了几天的时间，收获也很多，也很感谢前人的经验，让我少走了很多弯路。"世上无难事，只怕有心人"，自己真正尝试后也并没有想象中的那么难，最重要的就是勇敢的踏出第一步！

最后附上编译后的ijkplayer代码[ijkplayer传送门](https://github.com/jsyjst/ijkplayer)

> 参考博客：
>
> - [ijkplayer编译so库真没那么难](https://blog.csdn.net/coder_pig/article/details/79134625?depth_1-utm_source=distribute.pc_relevant.none-task&utm_source=distribute.pc_relevant.none-task)
> - [ijkplayer编译（Ubuntu + Win双系统）](https://blog.csdn.net/qq_27582179/article/details/51955794?depth_1-utm_source=distribute.pc_relevant.none-task&utm_source=distribute.pc_relevant.none-task)
> - [解决vmtools已安装但还是不能共享文件、复制粘贴](https://blog.csdn.net/zerolity/article/details/81206476?depth_1-utm_source=distribute.pc_relevant.none-task&utm_source=distribute.pc_relevant.none-task)
> - [Ijkplayer demo 基本使用](https://www.jianshu.com/p/5d1d46aa721d)