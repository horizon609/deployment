---
layout: post
title:  "通过轮子LabelsView，来巩固Veiw的绘制的知识"
date:   2018-07-20 21:26:43
categories: jekyll update
comments: true
---
最近，我发现项目里有好多用到标签的需求。于是不由地想：如果把这些散落在各处的需求用一个统一化的手段处理多好。而且我最近又在复习View绘制的机制，很想动手写一个轮子来定制符合美团管家特定需求的轮子。但是无奈项目工期所迫，我决定还是找一个现成的轮子改一改把它用起来更符合实际情况。
<!--break-->
于是我上网看了一下，选了LabelsView这个项目。因为需求并没有那么庞大，选那种花哨复杂的会无形增加项目的代码量。并且太多复杂特性会削弱对于核心特性的关注度。所以选择了LabelsView。它已经完全满足项目当前的需求了。

先来回顾一下View绘制的过程吧。

大体来说，可以把android界面ui的结构想象成一棵树。ViewRootImpl是树根，也就是最上层的，绘制起始的地方；再下一层是DecorView；再下层是各种自己实现的ViewGroup；最后是叶子View。经过测量、布局、绘制三大流程，最终把页面全部展现在用户面前。

一切的开始从ViewRootImpl#performTraversals说起，三大流程都是从这个方法起始的，用一段伪码表示就是
{% highlight java %}
private void performTraversals() {
...
performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
...
performLayout(lp, mWidth, mHeight);
...
performDraw();
..
}
{% endhighlight %}
当这个performTraversals()方法执行完毕之后，整个View的绘制流程就完毕了。那么具体这三大方法是如何传递下去的呢？

首先来分析performMeasure，先看看childWidthMeasureSpec和childHeightMeasureSpec两个参数，代表什么含义？往上看代码
{% highlight java %}
int childWidthMeasureSpec = getRootMeasureSpec(mWidth, lp.width);
int childHeightMeasureSpec = getRootMeasureSpec(mHeight, lp.height);
{% endhighlight %}
就是通过一个getRootMeasureSpec算法，将mWidth和lp.width转化成childWidthMeasureSpec，那么先明确一下mWidth和mHeight，分别是屏幕的宽高像素。而lp指的是WindowManager.LayoutParams lp = mWindowAttributes，这个变量指的是当前Window的LayoutParams。Window内的flag很多，比如Activity的Window就是全屏，而Dialog则不是。那么lp.width代表的就是Window中设置的宽高。

具体来看一下算法getRootMeasureSpec：
{% highlight java %}
private static int getRootMeasureSpec(int windowSize, int rootDimension) {
        int measureSpec;
        switch (rootDimension) {

        case ViewGroup.LayoutParams.MATCH_PARENT:
            // Window can't resize. Force root view to be windowSize.
            measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.EXACTLY);
            break;
        case ViewGroup.LayoutParams.WRAP_CONTENT:
            // Window can resize. Set max size for root view.
            measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.AT_MOST);
            break;
        default:
            // Window wants to be an exact size. Force root view to be that size.
            measureSpec = MeasureSpec.makeMeasureSpec(rootDimension, MeasureSpec.EXACTLY);
            break;
        }
        return measureSpec;
    }
{% endhighlight %}
MeasureSpec.makeMeasureSpec的过程是通过屏幕大小或window大小及window的SpecMode来算出来的一个叫做measureSpec的int值，它包含了SpecSize,SpecMode两部分。

specMode有三种：UNSPECIFIED；AT_MOST；EXACTLY。AT_MOST代表的是在xml中android:layout_width="wrap_content",而EXACTLY则代表的是android:layout_width="match_parent"&&android:layout_width="60dp"这种具体数值。这两种模式其实都是子View和父容器View共同决定测量的，而UNSPECIFIED则和父容器无关，单纯由子View决定。UNSPECIFIED方式测量对于应用开发者来说几乎没有用到，所以忽略。

而在得出了measureSpec时，就完成了当前DecorView的测量准备工作。接着把参数通过mView.measure(childWidthMeasureSpec, childHeightMeasureSpec)传递到DecorView中。measure(int,int)是View中的方法。而measure会调用View#onMeasure(int, int).这里需要注意的是：测量的过程是由View来驱动的，最后是由View#onMeasure落实的。

而DecorView extends FrameLayout extends ViewGroup extends View 

DecorView#onMeasure的伪码如下
{% highlight java %}
@Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    	  //处理了AT_MOST模式下的widthMeasureSpec和heightMeasureSpec（后续揭示为啥只针对AT_MOST）
    		super.onMeasure(widthMeasureSpec, heightMeasureSpec);
        ...
    }
{% endhighlight %}
FrameLayout#onMeasure
{% highlight java %}
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
	//通过子Views去测算maxWidth及maxHeight
  ..
  //设置DecorView自己的测量值
   setMeasuredDimension(resolveSizeAndState(maxWidth, widthMeasureSpec, childState),
                resolveSizeAndState(maxHeight, heightMeasureSpec,
                        childState << MEASURED_HEIGHT_STATE_SHIFT));
                        count = mMatchParentChildren.size();
  //遍历测量子View的MeasureSpec                      
        if (count > 1) {
            for (int i = 0; i < count; i++) {
                final View child = mMatchParentChildren.get(i);
                final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();

                final int childWidthMeasureSpec;
                if (lp.width == LayoutParams.MATCH_PARENT) {
                //若子View的lp是MATCH_PARENT，则子View的width由DecorView的mMeasuredWidth和DecorView的padding和自身的
                //margin值共同决定，子View的sepcMode是EXACTLY
                    final int width = Math.max(0, getMeasuredWidth()
                            - getPaddingLeftWithForeground() - getPaddingRightWithForeground()
                            - lp.leftMargin - lp.rightMargin);
                    childWidthMeasureSpec = MeasureSpec.makeMeasureSpec(
                            width, MeasureSpec.EXACTLY);
                } else {
                    childWidthMeasureSpec = getChildMeasureSpec(widthMeasureSpec,
                            getPaddingLeftWithForeground() + getPaddingRightWithForeground() +
                            lp.leftMargin + lp.rightMargin,
                            lp.width);
                }

                final int childHeightMeasureSpec;
                if (lp.height == LayoutParams.MATCH_PARENT) {
                    final int height = Math.max(0, getMeasuredHeight()
                            - getPaddingTopWithForeground() - getPaddingBottomWithForeground()
                            - lp.topMargin - lp.bottomMargin);
                    childHeightMeasureSpec = MeasureSpec.makeMeasureSpec(
                            height, MeasureSpec.EXACTLY);
                } else {
                    childHeightMeasureSpec = getChildMeasureSpec(heightMeasureSpec,
                            getPaddingTopWithForeground() + getPaddingBottomWithForeground() +
                            lp.topMargin + lp.bottomMargin,
                            lp.height);
                }
 //将子View测量好的MeasureSpec进行更下一层的测量
                child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
            }
        }
}
{% endhighlight %}
最终完成测量的是调用方法View#setMeasuredDimension
{% highlight java %}
protected final void setMeasuredDimension(int measuredWidth, int measuredHeight) {
        boolean optical = isLayoutModeOptical(this);
        if (optical != isLayoutModeOptical(mParent)) {
            Insets insets = getOpticalInsets();
            int opticalWidth  = insets.left + insets.right;
            int opticalHeight = insets.top  + insets.bottom;

            measuredWidth  += optical ? opticalWidth  : -opticalWidth;
            measuredHeight += optical ? opticalHeight : -opticalHeight;
        }
        setMeasuredDimensionRaw(measuredWidth, measuredHeight);//在此设置Veiw的mMeasuredWidth和mMeasuredHeight变量
    }
{% endhighlight %}
子View在!LayoutParams.MATCH_PARENT的情况下是怎么测量measureSpec的呢？ViewGroup#getChildMeasureSpec
{% highlight java %}
public static int getChildMeasureSpec(int spec, int padding, int childDimension) {
        int specMode = MeasureSpec.getMode(spec);
        int specSize = MeasureSpec.getSize(spec);

        int size = Math.max(0, specSize - padding);

        int resultSize = 0;
        int resultMode = 0;

        switch (specMode) {
        // Parent has imposed an exact size on us
        case MeasureSpec.EXACTLY:
            if (childDimension >= 0) {
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                // Child wants to be our size. So be it.
                resultSize = size;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                // Child wants to determine its own size. It can't be
                // bigger than us.
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            }
            break;

        // Parent has imposed a maximum size on us
        case MeasureSpec.AT_MOST:
            if (childDimension >= 0) {
                // Child wants a specific size... so be it
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                // Child wants to be our size, but our size is not fixed.
                // Constrain child to not be bigger than us.
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                // Child wants to determine its own size. It can't be
                // bigger than us.
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            }
            break;

        // Parent asked to see how big we want to be
        case MeasureSpec.UNSPECIFIED:
            if (childDimension >= 0) {
                // Child wants a specific size... let him have it
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                // Child wants to be our size... find out how big it should
                // be
                resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
                resultMode = MeasureSpec.UNSPECIFIED;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                // Child wants to determine its own size.... find out how
                // big it should be
                resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
                resultMode = MeasureSpec.UNSPECIFIED;
            }
            break;
        }
        //noinspection ResourceType
        return MeasureSpec.makeMeasureSpec(resultSize, resultMode);
    }
{% endhighlight %}
总结一下：

当子View的childDimension值>=0时，子View的resultMode是EXACTLY，resultSize是lp值的大小

当子View的childDimension值是WRAP_CONTENT时，子View的resultMode是AT_MOST；resultSize是父View的大小

当子View的childDimension值是MATCH_PARENT，父View的SpecMode是AT_MOST时，子View的resultMode是AT_MOST；父View的SpecMode是EXACTLY时，子View的resultMode是EXACTLY；子View的resultSize是父View的大小。(随父View)

再看看普通View中默认怎么样实现View#onMeasure
{% highlight java %}
getDefaultSizeprotected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
                getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
    }
{% endhighlight %}
在这个过程中发生了值的一系列转化，具体看看ViewGroup#getDefaultSize方法
{% highlight java %}
public static int getDefaultSize(int size, int measureSpec) {
        int result = size;
        int specMode = MeasureSpec.getMode(measureSpec);
        int specSize = MeasureSpec.getSize(measureSpec);

        switch (specMode) {
        case MeasureSpec.UNSPECIFIED:
            result = size;
            break;
        case MeasureSpec.AT_MOST:
        case MeasureSpec.EXACTLY:
            result = specSize;
            break;
        }
        return result;
    }
{% endhighlight %}
看上述的代码，非UNSPECIFIED情况下，求值过程和输入的参数size没关系。所以很大程度上，measureSpec就代表了转化后的值，只有UNSPECIFIED情况下，用到了size，那么size是什么？可以看ViewGroup#getSuggestedMinimumWidth()
{% highlight java %}
protected int getSuggestedMinimumHeight() {
    return (mBackground == null) ? mMinHeight : max(mMinHeight, mBackground.getMinimumHeight());
 }
{% endhighlight %}
代码已经很清楚了。

通过getDefaultSize和getChildMeasureSpec都揭示了一件事：就是在自定义View#onMeasure的过程中需要重写specMode是AT_MOST的情况，否则specSize是父View的大小

目前为止已经分析完了DecorView和普通View的测量过程，而对于ViewGroup的分析，我打算分析LabelsView#onMeasure来说明ViewGroup的测量过程

代码如下：
{% highlight java %}
@Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
				int count = getChildCount();
        int maxWidth = MeasureSpec.getSize(widthMeasureSpec) - getPaddingLeft() - getPaddingRight();

        int contentHeight = 0; //记录内容的高度
        int lineWidth = 0; //记录行的宽度
        int maxLineWidth = 0; //记录最宽的行宽
        int maxItemHeight = 0; //记录一行中item高度最大的高度
        boolean begin = true; //是否是行的开头

        for (int i = 0; i < count; i++) {
            View view = getChildAt(i);
            measureChild(view, widthMeasureSpec, heightMeasureSpec);

            if (!begin) {
                lineWidth += mWordMargin;
            } else {
                begin = false;
            }

            if (maxWidth <= lineWidth + view.getMeasuredWidth()) {
                contentHeight += mLineMargin;
                contentHeight += maxItemHeight;
                maxItemHeight = 0;
                maxLineWidth = Math.max(maxLineWidth, lineWidth);
                lineWidth = 0;
                begin = true;
            }
            maxItemHeight = Math.max(maxItemHeight, view.getMeasuredHeight());

            lineWidth += view.getMeasuredWidth();
        }

        contentHeight += maxItemHeight;
        maxLineWidth = Math.max(maxLineWidth, lineWidth);

        setMeasuredDimension(measureWidth(widthMeasureSpec, maxLineWidth),
                measureHeight(heightMeasureSpec, contentHeight));
    }    
{% endhighlight %}

{% highlight java %}
private int measureWidth(int measureSpec, int contentWidth) {
        int result = 0;
        int specMode = MeasureSpec.getMode(measureSpec);
        int specSize = MeasureSpec.getSize(measureSpec);

        if (specMode == MeasureSpec.EXACTLY) {
            result = specSize;
        } else {
            result = contentWidth + getPaddingLeft() + getPaddingRight();
            if (specMode == MeasureSpec.AT_MOST) {
                result = Math.min(result, specSize);
            }
        }
        result = Math.max(result, getSuggestedMinimumWidth());
        return result;
    }
{% endhighlight %}

简单的概括：LabelView支持精确模式和最大模式，精确模式按用户自己写的宽高来测量；而最大模式则由onMeasure通过addLables的个数测算而得。
