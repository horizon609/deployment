---
layout: post
title:  "替换android系统签名的方法"
date:   2016-3-2 11:52:21
categories: jekyll update
comments: true
---
##研究背景
上一篇文章正向提供了一个思路：将一个第三方app变成一个系统级权限的app。而这片文章从另外一个角度出发，将rom中现有system级别的app换成自己的签名，再将自己的第三方app做同样的签名放进system/app下，同样实现了第三方app获取系统权限的目的。
<!--break-->

##整体思路
前提：手机已经root
通过将rom中具有系统权限的app进行重签名，从而以后直接push进/system/app的应用只要具有自己的前面就可以获取系统权限。

##实现步骤

### 1. 取出data/system中的packages.xml文件，查找关键字`sharedUserId="1000"`，如下所示

{% highlight ruby %}
<package name="com.android.huawei.projectmenu" codePath="/system/app/ProjectMenuAct.apk" nativeLibraryPath="/data/app-lib/ProjectMenuAct" flags="572997" ft="1549f2cd0b8" it="1549f2cd0b8" ut="1549f2cd0b8" version="8" `sharedUserId="1000"`>
    <sigs count="1">
        <cert index="0" />
    </sigs>
    <signing-keyset identifier="1" />
</package>
{% endhighlight %}

###2. 通过sharedUserId="1000"获取所有的包名，将system/app和framework中的这些部分apk拿出来进行重签名，之后在打包替换回去。

###3. 重启手机

##原理
每次启动android操作系统会把这些系统应用随机排序一次，把遇到的第一个系统应用放在index=1的位置上。如果只是重签名了第一个位置上的系统应用，那么它的签名是改了。可是下次它的位置变了，然后系统发现它和其他的系统app签名不匹配会把它直接删除。只能把所有的系统应用都重签一次。
