# Android 热修复  #

从2015年开始，为了解决app突然出现的bug需要快速动态修复，但又不想重新打包、测试、发布、重现安装等一些问题，Android开发领域里对热修复技术的探讨越来越多，包括支付宝、淘宝、微信、QQ空间、饿了么、美丽说蘑菇街、美团大众点评等团队都相继推出了自己的热修复方案。这些方案看着实现方式都有几种，但从总体上来说，比较热门的是 Dexposed、AndFix、ClassLoader、Instant Run 和微信的 Tinker 几种热修复方案。

那么我们就首先来了解一下这几种方案，并对它们进行分析和比较。

## Dexposed ##

Github地址：[https://github.com/alibaba/dexposed](https://github.com/alibaba/dexposed)

### 原理 ###

Dexposed 是由Alibaba 团队开发的。是基于Xposed的AOP（面向切面编程）框架，方法级粒度，可以进行AOP编程、插桩、热补丁、SDK hook等功能。

在Dalvik虚拟机下，主要是通过改变一个方法对象方法在Dalvik虚拟机中的定义来实现，具体做法就是将该方法的类型改变为Native并且将这个方法的实现链接到一个通用的Native Dispatch方法上。这个 Dispatch方法通过JNI回调到Java端的一个统一处理方法，最后在统一处理方法中调用before, after函数来实现AOP。在Art虚拟机上目前也是通过改变一个 ArtMethod的入口函数来实现。具体实现方式我们会在后面的文章介绍到。

### 应用场景 ###

- AOP编程
- 仪表（用于测试，性能监测等）
- 在线热补丁修复
- SDK hooking 以提供更好的开发体验

### 系统支持 ###

|  Runtime 	| Android Version  |	Support |
|:----------|---------:|-------:|
| Dalvik 	|  2.2     |	Not Test |
| Dalvik 	|  2.3 	   |    Yes
| Dalvik 	|  3.0     |	No
| Dalvik 	|  4.0-4.4 |    Yes
| ART 	    |  5.0 	   |    Testing
| ART 	    |  5.1     |	No
| ART 	    |  M 	   |    No

### 优点： ###

1. 无侵入性，性能损耗较小
2. 即时生效 

### 缺点： ###

1. 不支持ART模式(Android Runtime)
2. 需要反射写混淆后的代码
3. 粒度太细，要替换的方法多的话，工作量会比较大

## AndFix ##

Github地址：[https://github.com/alibaba/AndFix](https://github.com/alibaba/AndFix)

### 原理 ###

AndFix也是由Alibaba 团队开发的。AndFix的原理就是方法的替换，把有bug的方法替换成补丁文件中的方法。 
AndFix采用native hook的方式，这套方案直接使用dalvik_replaceMethod替换class中方法的实现。由于它并没有整体替换class, 而field在class中的相对地址在class加载时已确定，所以AndFix无法支持新增或者删除filed的情况。基本原理图如下：

![](http://i.imgur.com/afX9Ku3.png)

### 优点： ###
    
1. 立即生效
2. 补丁较小
3. 安全性高，对应用无侵入，几乎无性能损耗
4. 支持全平台
5. 项目成熟，文档健全。

### 缺点： ###

1. 兼容性不好
2. 开发不透明
3. 不支持新增字段，以及修改<init>方法，也不支持对资源的替换
4. 需要使用加固前的apk制作补丁，但是补丁文件很容易被反编译，也就是修改过的类源码容易泄露
5. 使用加固平台可能会使热补丁功能失效

## ClassLoader ##

ClassLoader没有开源出来，但是可以参照开源出来的Nuwa。

Github地址：[https://github.com/jasonross/Nuwa](https://github.com/jasonross/Nuwa)

### 原理 ###

通过DexClassLoader对象，动态加载补丁dex，再通过反射将补丁dexdex插入到dexElements最前面。要实现热更新，需要热更新的类要防止被打上ISPREVERIFIED标记。

### 优点： ###
    
1. 项目成熟，文档健全
2. 集成简单
3. 支持添加新加类和新的字段
4. 兼容性和稳定性高

### 缺点： ###

1. 支持gradle1.5以下
2. 需要应用重启后生效

## Instant Run  ##

[http://www.jianshu.com/p/2e23ba9ff14b](http://www.jianshu.com/p/2e23ba9ff14b)

### 原理 ###
Instant Run，是android studio2.0新增的一个运行机制，在你编码开发、测试或debug的时候，它都能显著减少你对当前应用的构建和部署的时间。
第一次点击run、debug按钮的时候，它运行时间和我们往常一样。但是接下去的时间里，你每次修改代码后点击run、debug按钮，对应的改变将迅速的部署到你正在运行的程序上，传说速度快到你都来不及把注意力集中到手机屏幕上，它就已经做好相应的更改。

在实践中，这意味着：

1. 只对代码改变部分做构建和部署
2. 不重新安装应用
3. 不重启应用
4. 不重启activity

![](http://i.imgur.com/HpN3ERM.png)

也就是Manifest文件合并、打包，和res一起被AAPT合并到APK中，同样项目代码被编译成字节码，然后转换成.dex 文件，也被合并到APK中。

### 优点 ###
1. 更新补丁启动方式：instant-run根据补丁文件后缀提供了3种启动方式（热启动、温启动、冷启动）更加方便更新补丁
2. Application打补丁：instant-run实现了application替换
3. 资源的替换：instant-run是资源目录的替换和操作resources.ap_文件来实现，对资源替换支持比较好。

### 缺点 ###

1. Instant Run是不能回退的。代码更改可以通过热拔插快速部署，但是热拔插会影响应用的初始化，需要重启应用。
2. 如果应用的minSdkVersion小于21，可能多数的Instant Run功能会挂掉，这里提供一个解决方法，通过product flavor建立一个minSdkVersion大于21的新分支，用来debug。
3. Instant Run目前只能在主进程里运行，如果应用是多进程的，类似微信，把webView抽出来单独一个进程，那热、温拔插会被降级为冷拔插。
4. 在Windows下，Windows Defender Real-Time Protection可能会导致Instant Run挂掉。
5. 暂时不支持Jack compiler，Instrumentation Tests，或者同时部署到多台设备。

## 微信的 Tinker ##

Github地址：[https://github.com/WeMobileDev/article/blob/master/%E5%BE%AE%E4%BF%A1Android%E7%83%AD%E8%A1%A5%E4%B8%81%E5%AE%9E%E8%B7%B5%E6%BC%94%E8%BF%9B%E4%B9%8B%E8%B7%AF.md#rd](https://github.com/WeMobileDev/article/blob/master/%E5%BE%AE%E4%BF%A1Android%E7%83%AD%E8%A1%A5%E4%B8%81%E5%AE%9E%E8%B7%B5%E6%BC%94%E8%BF%9B%E4%B9%8B%E8%B7%AF.md#rd)

### 原理 ###

通过新旧apk比较，使用gradle从插件生成.dex补丁文件（并不是真正的dex文件），补丁通过服务器下发后尝试对dex文件二路归并进行合并，最终生成全量的dex文件，与生成补丁互为逆过程，生成全量dex文件后进行optimize操作，最终生成odex文件。在Application中进行反射调用已经合成的dex文件。

### 优点 ###

1. 合成整包，不用在构造函数插入代码，防止verify，verify和opt在编译期间就已经完成，不会在运行期间进行
2. 性能提高。兼容性和稳定性比较高
3. 开发者透明，不需要对包进行额外处理

### 缺点 ###

1. 需要应用重启后生效
2. 需要给应用开启新的进程才能进行合并，并且很容易因为内存消耗等原因合并失败
3. 合并时占用额外磁盘空间，对于多DEX的应用来说，如果修改了多个DEX文件，就需要下发多个patch.dex与对应的classes.dex进行合并操作时这种情况会更严重，因此合并过程的失败率也会更高。









