---
layout: post
title:  "Can not perform this action after onSaveInstanceState"
date:   2017-08-18s 17:15:43
categories: jekyll update
comments: true
---

## 产生原因

针对不同的android版本，发生crash的契机略有区别。安卓4.2.2比6.0.1发生的可能性更大。在pos项目里。在监控crash上报的时候发现在android6.0.1上只发现过提示activity onDestroy的crash。
<!--break-->

产生原因就是app被系统销毁的时机是不确定的，当销毁之后收到了被踢下线的通知，然后显示dialogFragment，必然会崩溃。而更容易发生crash的是在android4.2.2上。在点击home之后，收到显示dialogFragment，崩溃，提示Can not perform this action after onSaveInstanceState。综合上述两种情况，归纳其产生原因是：**在依赖的FragmentActivity调用onStop()之后调用DialogFragment.show()。**

## 时序考虑

###1. 在onsave内，调用dialogFragment——不会发生崩溃

###2. 在onsave后，activity销毁前，调用dialogFragment——

####a. 利用onSaveInstanceState保存fragment现场，等到Activity进行到onCreate的时候恢复fragment现场（防止系统直接销毁Activity）

####b. 保存现场到activity的成员变量当中（常用于系统并没有杀死Activity，但是用户点击home键会到系统桌面），在回应用后Activity调用onResume时恢复fragment现场

#### ps：以上两点是保证fragment在activity不可见后又可见的情况下能正常显示的两个手段，理论上b比a的场景更长久

那么需要考虑的问题就是两点：

###1. 在onSaveInstanceState中保存fragment的状态

###2. 在当事Activity的回调期间or发消息者和当事Activity通信的过程中（例如eventbus）保存fragment的状态

由上述时序考虑得出最优解决方案，也是我目前在pos中所做的重构：
> * 需要立一个flag：mIsStateAlreadySaved。用来记录当事Activity的状态：是不是调用了onSaveInstanceState
{% highlight java %}
@Override
protected void onResume() {
    super.onResume();
    mIsStateAlreadySaved = false;
}
{% endhighlight %}

{% highlight java %}
@Override
protected void onStop() {
    mIsStateAlreadySaved = true;
    super.onSaveInstanceState();
}
{% endhighlight %}
> * 在通知端建立一个callback的interface，方便注册的当事activity监听发出显示fragment的事件
{% highlight java %}
public interface BusinessExceptionCallback{
    void updatekickoffPage(Message msg);
}
 
public void registerCallback(BusinessExceptionCallback callback) {
    mCallback = callback;
}
 
public void unregisterCallback(BusinessExceptionCallback callback) {
    if (callback == mCallback) {
        mCallback = null;
    }
}
{% endhighlight %}
> * 当时Activity中，注册监听，反注册监听
{% highlight java %}
@Override
public void onCreate(Bundle icicle) {
    BusinessExceptionHandler.getInstance().registerCallback(this);
}
  
@Override
protected void onDestroy(Bundle outState) {
    //防止内存泄漏
    BusinessExceptionHandler.getInstance().unregisterCallback(this);
}
{% endhighlight %}
> * 通知端，发出通知
{% highlight java %}
/**
 * handler 处理被踢下线的逻辑
 */
private class DialogHandler extends Handler {
 
    DialogHandler() {
        super(Looper.getMainLooper());
    }
 
    @Override
    public void handleMessage(Message msg) {
        super.handleMessage(msg);
        if (msg.obj == null || needReturn()) {
            return;
        }
        if (msg.what == KICKED_OFF && msg.obj instanceof ApiResponse) {
            //自己被踢下线了，立刻关闭配置时间戳广播，避免打扰副POS
            MessageDataHelper.getInstance().stopAllTask();
            ELog.d(TAG, "prepare to send kick off msg");
            if(mCallback != null){
                mCallback.updatekickoffPage(msg);
            }
        }
        ....
    }
}
{% endhighlight %}
> * 当事Activity,处理更新界面
{% highlight java %}
public class MainActivity extends BasePosActivity implements MainPageContract.View, BusinessExceptionHandler.BusinessExceptionCallback{
    @Override
    public void updatekickoffPage(Message msg) {
        if (!mIsStateAlreadySaved) {
            mIShowDialog.showKickOffDialog(msg);
            removeCurrentFragment();
            if (mMasterPosInfoReceiveHelper != null) {
                mMasterPosInfoReceiveHelper.quit();
            }
            if (mMainPosInfoMulticast != null) {
                mMainPosInfoMulticast.quit();
            }
        } else {
            ELog.d(TAG, "saved! save status!");
            mKickOffFragmentMessage.copyFrom(msg);//这里注意，保存msg的状态，以便后续MainActivity重建时在onResume中恢复fragment
        }
  
    @Override
    protected void onResume() {
        super.onResume();
        ELog.d(TAG, "register callback");
        if (mKickOffFragmentMessage != null && mKickOffFragmentMessage.obj != null) {
            ELog.d(TAG, "prepare to show dialog onResume");
            mIShowDialog.showKickOffDialog(mKickOffFragmentMessage);
            removeCurrentFragment();
            mKickOffFragmentMessage = null;
        }
    }
}
{% endhighlight %}
> * 理论上还需要处理时序里的第三种情况，即当事Activity被销毁后的保存和恢复
{% highlight java %}
@Override
protected void onStop(Bundle outState) {
    mIsStateAlreadySaved = true;
    outState.putParcelable(KEY_SAVE_STATE_KICK_OFF_FRAGMENT, mKickOffFragmentMessage);
}
  
@Override
public void onCreate(Bundle icicle) {
    if (icicle != null && icicle.containsKey(KEY_SAVE_STATE_KICK_OFF_MESSAGE)) {
        mKickOffFragmentMessage = icicle.getParcelable(KEY_SAVE_STATE_KICK_OFF_MESSAGE);
        updatekickoffPage(mKickOffFragmentMessage);
        mKickOffFragmentMessage = null;
        isReCreate = true;
    }
     
}
{% endhighlight %}
> * 但是上述过程没有在pos的代码里体现，原因是pos中每次新建MainActivity都会请求一次发起状态，从而通过BusinessExceptionHandler发出显示fragment通知


