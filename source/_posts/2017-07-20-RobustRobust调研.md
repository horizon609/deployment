---
layout: post
title:  "Robust调研"
date:   2017-08-18s 17:15:43
categories: jekyll update
comments: true
---
## 何时加载

> * 越早加载越好，可以修复发生在靠前的bug。一般放在Application的onCreate里；
> * 注意事项：使用**multidex**的项目，需要在dex全部加载完后执行补丁加载；
<!--break-->

## 禁止某个patch的方式

需要在服务端把该条补丁删除，下次打补丁的时候，即便本地有这条被移除的补丁，也不会执行它。感觉这种方式不太灵活

## 补丁、插桩优先级顺序

{% highlight java %}
if(hasPatch){
    patch execute;
    other Extension ignore;
    notify listeners;
} else {
    for(extension:extensions) {
        execute extension;
        if(extension.isSupport){
            notify other extensions : extension is executed~
            break;
        }
    }
}
{% endhighlight %}

## 对于返回this，支持而不友好

对于方法的返回值是this的情况现在支持不好，比如builder模式，但在制作补丁代码时，可以通过如下方式来解决，增加一个类来包装一下(如下面的B类)
{% highlight java %}
method a(){
  return this;
}
{% endhighlight %}
改为
{% highlight java %}
method a(){
  return new B().setThis(this).getThis();
}
{% endhighlight %}

## 缺点

目前**更改构造函数**和**新增字段**都在内测阶段，所以无法实现。但是也有不优雅的解决方案：**增加一个新类**
