---
layout: post
title:  "Sublime Text撸android源码的一些姿势"
date:   2017-08-18s 17:15:43
categories: jekyll update
comments: true
---
之前在windows上撸源码都是用source insight。编译完源码后直接引入impl就可以直接看了，跳转通过ctrl+鼠标左键就完成了。其他操作也类似于eclipse，十分愉快。在mac上找了个替代方案。最近刚开始用。初步感觉还不错。记录下配置和基本使用姿势。
<!--break-->

首先，你需要自备aosp源码、下载sublime text3（废话）然后直接用st的file->open工程目录。这个时候就是一堆源代码，没有任何跳转关系。这么看起来很不爽，那么我们需要装个插件——Ctags。这个可以帮我们管理所有跳转的关系，到时候直接点进去就可以看方法调用。在安装Ctags之前，先来安装一下Package Control这个插件。

## 安装Package Control

###1. 打开控制台：view->showConsole

###2. 控制台输入一下代码并回车
{% highlight java %}
import urllib.request,os; pf = 'Package Control.sublime-package'; ipp = sublime.installed_packages_path(); urllib.request.install_opener( urllib.request.build_opener( urllib.request.ProxyHandler()) ); open(os.path.join(ipp, pf), 'wb').write(urllib.request.urlopen( 'http://sublime.wbond.net/' + pf.replace(' ','%20')).read())
{% endhighlight %}

###3. 等一会，重启ST，然后到Preferences里面看一下是否有Package Control，有就成功了。

## 安装Ctags

###1. preferences→package control:输入ctags并且选择安装

###2. 这时你在打开的文件中，右键菜单中会多一个Navigate to Definition菜单项，在侧左栏的工程/项目文件上右键会看到CTags: Rebuild Tags菜单项

###3. 这时右键工程选择ctags:rebuild tags，会得到类似下面一个报错
![](http://pahdlppe7.bkt.clouddn.com/Sublime:error.jpeg)

因为这时路径不对，可以重新安装配置一下。具体如下：
{% highlight java %}
tlhunter@mac:~ $ mkdir ctags; cd ctags
tlhunter@mac:~/ctags $ curl -OL http://www.thomashunter.name/wp-content/uploads/ctags-5.8.tar.gz
tlhunter@mac:~/ctags $ tar xzvf ctags-5.8.tar.gz
tlhunter@mac:~/ctags $ cd ctags-5.8
tlhunter@mac:~/ctags/ctags-5.8 $ ./configure
tlhunter@mac:~/ctags/ctags-5.8 $ make
tlhunter@mac:~/ctags/ctags-5.8 $ sudo make install
{% endhighlight %}

###4. 这个时候就不会报上述错误了。然后来改点TS里的ctags的配置：
打开菜单在Preferences(设置)菜单中打开Package Control(插件管理器)settings->ctags->settings-user和settings-default，复制所有default里的内容到user里，然后更改user里的command配置项里的内容为：/usr/local/bin/ctags

###5. 侧左栏的工程/项目文件上右键执行CTags: Rebuild Tags菜单项，就可以生成.tags文件，说明可以正常工作了

###6. 默认的点击方法跳转的快捷键是ctrl+shift+鼠标左键，我们可以将其修改了，我们把Mouse Binding default中的配置全部复制到user中，然后修改modifiers中的属性，将其全部改为command。以后就可以靠commander这个快捷键一统天下了唔噗噗噗~

## 一些快捷键 

Command+P：查找文件

Command+右键：返回文件或函数跳转的原始位置

Command+shift+f: 全局搜索

Command+左键：文件或函数跳转

Command+R：查找方法

## 启用vim模式
{% highlight java %}
 "ignored_packages":
　[
　]
{% endhighlight %}

## 参考链接

### [](http://yupengt66y.wang/2016/03/17/SublimeText-AndroidSourceConfig/http://jingyan.baidu.com/article/48206aeafba820216ad6b3f5.html)

### [Installing VIM Tagbar with MacVIM in OS X](https://thomashunter.name/blog/installing-vim-tagbar-with-macvim-in-os-x/)
