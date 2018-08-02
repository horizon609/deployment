---
layout: post
title:  "notification适配"
date:   2018-01-10 15:50:43
categories: jekyll update
comments: true
---
## [如何显示一个通知？](https://developer.android.com/guide/topics/ui/notifiers/notifications.html?hl=zh-cn)
<!--break-->

### 几个概念： [Notification](https://developer.android.com/reference/android/app/Notification.html)、[NotificationCompat.Builder](https://developer.android.com/reference/android/support/v4/app/NotificationCompat.Builder.html)、[Notification.Builder](https://developer.android.com/reference/android/app/Notification.Builder.html)

先说后两者：都是提供更方便的模板方法去获取通知内容、操作通知等。如果想适配api level 4以前的版本，需要用NotificationCompat.Builder。在目前的美团管家项目当中，这两者可谓没区别。他们比起Notification来说，后者创建通知更方便。Notification的构造函数在api level 11的时候废弃，因为这个版本加入了Notification.Builder

#### 创建通知
{% highlight java %}
NotificationCompat.Builder mBuilder =
        new NotificationCompat.Builder(this)
        .setSmallIcon(R.drawable.notification_icon)
        .setContentTitle("My notification")
        .setContentText("Hello World!");
// Creates an explicit intent for an Activity in your app
Intent resultIntent = new Intent(this, ResultActivity.class);

// The stack builder object will contain an artificial back stack for the
// started Activity.
// This ensures that navigating backward from the Activity leads out of
// your application to the Home screen.
TaskStackBuilder stackBuilder = TaskStackBuilder.create(this);
// Adds the back stack for the Intent (but not the Intent itself)
stackBuilder.addParentStack(ResultActivity.class);
// Adds the Intent that starts the Activity to the top of the stack
stackBuilder.addNextIntent(resultIntent);
PendingIntent resultPendingIntent =
        stackBuilder.getPendingIntent(
            0,
            PendingIntent.FLAG_UPDATE_CURRENT
        );
mBuilder.setContentIntent(resultPendingIntent);
NotificationManager mNotificationManager =
    (NotificationManager) getSystemService(Context.NOTIFICATION_SERVICE);
// mId allows you to update the notification later on.
mNotificationManager.notify(mId, mBuilder.build());
{% endhighlight %}

### 三者的选择？
表面上看NotificationCompat.Builder最一劳永逸，但是存在问题。在android O上使用官方示例代码，没有显示出通知，并且得到了以下Toast
![cmd-markdown-logo](http://pahdlppe7.bkt.clouddn.com/notification%E9%80%82%E9%85%8D_toast.png)
可以看出原因：android O里需要channel这个参数。查看文档，发现NotificationCompat.Builder(Context context)在api level 26被废弃了。取而代之的是NotificationCompat.Builder(Context context, String channelId)。也就是说如果使用NotificationCompat.Builder的时候，若要适配android O，必须要把v4包升级到26。这一点伴随着风险，并且在项目当中，代码有些地方引用低版本的v4包，升级后，需要重新适配。贸然升级会有诸多问题。于是转而选择了Notification.Builder。

### android O
加入了[NotificationChannel](https://developer.android.com/reference/android/app/NotificationChannel.html)。主要目的在于给通知分类，通过不同的channel id，可以让用户主动选择是否开启通知、是否允许通知震动等，以提升用户体验。

官方示例：https://github.com/googlesamples/android-NotificationChannels

[新的Action](https://developer.android.com/reference/android/provider/Settings.html):

ACTION_CHANNEL_NOTIFICATION_SETTINGS

ACTION_APP_NOTIFICATION_SETTINGS

## 通知跳转[TaskStackBuilder](https://developer.android.com/reference/android/support/v4/app/TaskStackBuilder.html?hl=zh-cn)
需求：当项目处于热启动状态，该项目发出通知，点击通知后跳转某个activity，点击back，需要返回该项目上一个activity。[官网解决方案](https://developer.android.com/guide/topics/ui/notifiers/notifications.html?hl=zh-cn#Progress)

#### back跳转回上一个activity
{% highlight java %}
...
Intent resultIntent = new Intent(this, ResultActivity.class);
TaskStackBuilder stackBuilder = TaskStackBuilder.create(this);
// Adds the back stack
stackBuilder.addParentStack(ResultActivity.class);
// Adds the Intent to the top of the stack
stackBuilder.addNextIntent(resultIntent);
// Gets a PendingIntent containing the entire back stack
PendingIntent resultPendingIntent =
        stackBuilder.getPendingIntent(0, PendingIntent.FLAG_UPDATE_CURRENT);
...
NotificationCompat.Builder builder = new NotificationCompat.Builder(this);
builder.setContentIntent(resultPendingIntent);
NotificationManager mNotificationManager =
    (NotificationManager) getSystemService(Context.NOTIFICATION_SERVICE);
mNotificationManager.notify(id, builder.build());
{% endhighlight %}

#### xml
{% highlight java %}
<activity
    android:name=".MainActivity"
    android:label="@string/app_name" >
    <intent-filter>
        <action android:name="android.intent.action.MAIN" />
        <category android:name="android.intent.category.LAUNCHER" />
    </intent-filter>
</activity>
<activity
    android:name=".ResultActivity"
    android:parentActivityName=".MainActivity">
    <meta-data
        android:name="android.support.PARENT_ACTIVITY"
        android:value=".MainActivity"/>
</activity>
{% endhighlight %}
上述跳转有一个缺点：需要指定parentActivity。可以通过获取当前的topActivity作为parentActivity。在美团管家项目中，统一使用SchemeUrl进行跳转。
