#Context研究笔记
android中有一个上帝类掌控着很多应用的基本信息,比如对资源文件进行访问的getResource或者getAssets访问Assert文件夹,获取包名的getPackageName,甚至于对本地File和database文件的访问,这个类就是Context上下文。
##Context的定义
关于Context的定义,它的代码里已经有很清楚的解释

Interface to global information about an application environment.  This is
an abstract class whose implementation is provided by
the Android system.  It allows access to application-specific resources and classes, as well as
up-calls for application-level operations such as launching activities,
broadcasting and receiving intents, etc.

应用环境信息的的全局接口,这是一个虚拟的类,被用来进行继承。它的方法实现都是由android系统来实现。它允许访问应用的特殊资源和类,还可以执行应用等级的上层调用方法,比如开启activity，广播和接收intent 等等。
