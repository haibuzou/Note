最近从eclipse迁移了一个老项目到Android Studio，迁移的原因嘛，当然是65535的问题。迁移的过程中从编译到打包，踩了一堆坑，在此记录一下，希望能帮到，同样被坑的同行

#如何导入
推荐使用直接导入ADT的方式来进行导入工作，这样会自动为你添加依赖，构建好代码的架构
![这里写图片描述](http://img.blog.csdn.net/20160510161338072)

选择好eclipse的工程和要导入的目标工程后，直接next -> finish 选项用默认选项
![这里写图片描述](http://img.blog.csdn.net/20160510161525317)

静静的祈祷，并等待导入结束
![这里写图片描述](http://img.blog.csdn.net/20160510161722749)

#开始慢慢填坑路
AS构建项目完成后，报错是肯定的，先来看看第一个错误
##命名不规范
 ![这里写图片描述](http://img.blog.csdn.net/20160510161857209)
 这个错误很好理解，命名不规范的问题，按照要求改就可以了

## .9不符合要求
.9不符合规范是一个很普遍的情况，eclipse对于.9的要求并不像 Android Studio 中那样严格
![这里写图片描述](http://img.blog.csdn.net/20160510162715371)

eclipse项目中经常有这种 .9，四边并没有都没有给像素
![这里写图片描述](http://img.blog.csdn.net/20160510163106307)

这里我的处理方式 就是重新画.9，给他加上边线
![这里写图片描述](http://img.blog.csdn.net/20160517085231026)

##libpng warning iCCP & libpng Error : Not a Png file
Not Png file 还好说找到文件格式不对的文件改后缀就可以了，但是如果项目中很多图片找不到怎么办呢? 后面再告诉你们。

还有libpng warning iCCP 就比较坑了，这个错误直接报到V7包里面的资源，问题的原因，我也比较费解。如果谁知道希望留言告知一下
![这里写图片描述](http://img.blog.csdn.net/20160510164228189)

我的解决方案就比较暴力了，在主项目的build.gradle 文件中做出如下配置
![这里写图片描述](http://img.blog.csdn.net/20160510164620897)
关键配置:     aaptOptions.cruncherEnabled = false     aaptOptions.useNewCruncher = false
就可以直接忽略掉libpng的 2个错误


##AndroidManifest.xml merge冲突
eclipse中经常使用的库工程，同样可以在Android Studio中使用。在eclispe中库工程的AndroidManifest.xml，与主工程的AndroidManifest.xml有相同的配置的时候是不会报错的，但是在Android Studio 中这些都是不允许的

### 权限的重复声明
需要删除重复的权限
![这里写图片描述](http://img.blog.csdn.net/20160510165345582)

###application 节点 重复的配置

allowBackup 重复设置
![这里写图片描述](http://img.blog.csdn.net/20160510165607208)

applcation 重复设置
![这里写图片描述](http://img.blog.csdn.net/20160510165711599)

解决方式有2种，删除一个工程的设置，一般是删除库工程，第二种就是与编译器的建议一样写tools : replace 

![这里写图片描述](http://img.blog.csdn.net/20160510170001524)


这里的tools 需要声明一下才能使用
![这里写图片描述](http://img.blog.csdn.net/20160510170047475)
 

## V4 包的错误
找不到类原因是V4包没有，但是我的ec项目中是有v4包的，至于导入后为什么没有了，我也不是很明白
 ![这里写图片描述](http://img.blog.csdn.net/20160510170632643)

build.gradle文件中设置依赖  ```   compile 'com.android.support:support-v4:23.2.1'```  就可以了



## 运行时内存不够的问题

![这里写图片描述](http://img.blog.csdn.net/20160510171709391)

这个错误其实是JVM的错误，加载的内容过多，内存不够。
解决方案，build.gradle中配置 
``` 
    dexOptions {
        javaMaxHeapSize "4g"
    }
```
这个配置一定要写在 android{} 结点里面

## 64K的问题
64k的问题直接采用的官方的方案，设置起来还是很简单的
添加依赖 :  ```compile 'com.android.support:multidex:1.0.1'```
修改Application
```java
	@Override
	protected void attachBaseContext(Context base) {
		super.attachBaseContext(base);
		MultiDex.install(this);
	}
```


最后到此我的项目就可以运行了  BUILD SUCCESS !!!  完结撒花~~~~
![这里写图片描述](http://img.blog.csdn.net/20160510173500757)



==========================================分割线

在我以为已经完工的时候，我尝试了一下打包，结果。。。 我再次懵逼了


# 打包的错误
我这里的打包时打签名的正式包，不是签名的话，下面的错误是不会出现的。

## layout.xml 文件中自定义属性错误
以前的项目中layout.xml自定义属性声明是直接跟包名的
```xmlns:app="http://schemas.android.com/apk/res/com.xxx"```
但是打包的时候会跟你报错，Adnroid Studio 中建议写成 res-auto
![这里写图片描述](http://img.blog.csdn.net/20160510173447333)
这种警告级别的东西，在打包时会成为错误爆出来


##Error: Expected resource of type styleable [ResourceType] 
这个错误就比较奇葩了，我的申明都是没问题的。而且在编译期间这个错误居然不会出现，
只有打包的时候才会出现，我又是吐了一口老血。所幸这个错误解决方案还是比较简单，
那就是直接过滤掉。在出问题的类上加上注解
```@SuppressWarnings("ResourceType")``` 就可以过滤掉了。


## ValidFragment错误
也是一个比较费解的错误
![这里写图片描述](http://img.blog.csdn.net/20160510175609669)
直接改吧，在fragment上给上注解 ``` @SuppressWarnings("ValidFragment") ``` 同样是过滤掉这个错误


## MissingTraslasion 错误
这个错误是在String.xml 里面没有设置是否能够被翻译的配置造成，但是我们应用一般不用给这个设置，也没见过报错啊。内鬼就出在友盟的工程中有设置 values-zh 导致我们自己的工程也被影响要设置翻译配置。
解决方案，直接在出问题的String.xml中添加配置 
```
<?xml version="1.0" encoding="utf-8"?>
<resources
  xmlns:tools="http://schemas.android.com/tools"
  tools:ignore="MissingTranslation" >
</resources>
```

HOlY SHIT!!!!! 
![这里写图片描述](http://img.blog.csdn.net/20160510180440157)


作者已经吐血了
 



