---
layout: post
title: 'Android桌面悬浮小窗的实现--FloatingView'
date: 2017-06-02
author: sung
cover: ''
tags: Android开发 悬浮窗 FloatingView
---

### 前言

做直播也一段时间了，也参考看过很多的不同的直播软件，有大公司的也有小公司的。千篇一律，唯独发现斗鱼熊猫有一个很细心的功能，视频悬浮窗口。在退出当前的直播间的时候会生成一个小的悬浮窗口继续播放。类似这种

![20170602185811484](https://github.com/ssccbb/ssccbb.github.io/blob/master/assets/cover/20170602185811484.png?raw=true)



### 思路

研究了一下之后使用WindowManager做出了同等效果。简述下原理，通过得到系统服务的WindowManager类，创建一个悬浮于桌面的小窗口，使用addview的方法添加自定义的播放view进悬浮窗口，重写窗口onscroll的方法计算x，y偏移量并更新view视图。



### 正文

播放的view根据个人需求添加，接下来详细讲解一个封装的悬浮view。首先自定义view继承的FrameLayout实现了GestureDetector的手势监听类，后面的ontouch用得到。

```java
/** 
 * Created by Administrator on 2016/9/13 
 * 跟随手指移动View以及显示/隐藏的接口封装 
 */  
public abstract class BaseFloatingView extends FrameLayout implements GestureDetector.OnGestureListener {}
```

构造方法

```java
public BaseFloatingView(Context context) {  
    this(context, null);  
}  
  
public BaseFloatingView(Context context, AttributeSet attrs) {  
    this(context, attrs, 0);  
}  
  
public BaseFloatingView(Context context, AttributeSet attrs, int defStyleAttr) {  
    super(context, attrs, defStyleAttr);  
    this.mContext = context;  
    this.mGestureDetector = new GestureDetector(context, this);  
}  
```

初始化/展示的方法封装。

上下文获取到系统的WindowManager类，new一个悬浮窗口需要的LayoutParams。

- TYPE_TOAST不检查permission
- TYPE_PHONE检查SYSTEM_ALERT_WINDOW权限
- FLAG_NOT_FOCUSABLE让window不能获得焦点，这样用户快就不能向该window发送按键事，即屏蔽掉onKey事件。
- 窗体设置了FLAG_WATCH_OUTSIDE_TOUCH这个flag，那么 用户点击窗体以外的位置时，将会在窗体的MotionEvent中收到MotionEvetn.ACTION_OUTSIDE事件。



```java
protected void showView(View view, int width, int height) {  
        mWindowManager = (WindowManager) mContext.getSystemService(Context.WINDOW_SERVICE);  
        //TYPE_TOAST仅适用于4.4+系统，假如要支持更低版本使用TYPE_SYSTEM_ALERT(需要在manifest中声明权限)  
        layoutParams = new WindowManager.LayoutParams(WindowManager.LayoutParams.TYPE_TOAST);  
        layoutParams.flags = WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE;  
        layoutParams.flags |= WindowManager.LayoutParams.FLAG_WATCH_OUTSIDE_TOUCH;  
        //layoutParams.flags |= WindowManager.LayoutParams.FLAG_LAYOUT_NO_LIMITS; //no limit适用于超出屏幕的情况，若添加此flag需要增加边界检测逻辑  
        layoutParams.width = width;  
        layoutParams.height = height;  
        layoutParams.format = PixelFormat.TRANSLUCENT;  
        mWindowManager.addView(view, layoutParams);  
    }  
```

接收要显示窗口的宽高addview添加view进WindowManager



隐藏窗口

```java
protected void hideView() {  
    if (null != mWindowManager)  
        mWindowManager.removeViewImmediate(this);  
    mWindowManager = null;  
}  

```



拖动的触摸事件
view按下的时候记录初始的位置x，y

```java
@Override  
public boolean onDown(MotionEvent e) {  
    lastX = e.getRawX();  
    lastY = e.getRawY();  
    return false;  
}  
```



在拖动的时候计算x，y方向上的偏移量，重新设置窗口的layoutparams，updateViewLayout更新view的位置，拖动的效果就出来了

```java
	@Override  
    public boolean onScroll(MotionEvent e1, MotionEvent e2, float distanceX, float distanceY) {  
        float nowX, nowY, tranX, tranY;  
        // 获取移动时的X，Y坐标  
        nowX = e2.getRawX();  
        nowY = e2.getRawY();  
        // 计算XY坐标偏移量  
        tranX = nowX - lastX;  
        tranY = nowY - lastY;  
        // 移动悬浮窗  
        layoutParams.x += tranX;  
        layoutParams.y += tranY;  
        //更新悬浮窗位置  
        mWindowManager.updateViewLayout(this, layoutParams);  
        //记录当前坐标作为下一次计算的上一次移动的位置坐标  
        lastX = nowX;  
        lastY = nowY;  
        return false;  
    }  
```

基础的拖动悬浮窗的封装就ok了



接下来实际使用的时候自定义view继承BaseFloatingView重写onSingleTapUp窗口单击事件暴露一个show的方法回掉basefloatingview的showview。所有需要在悬浮窗内显示的view，譬如视频播放就在自定义view内添加播放部分的布局以及播放的控制代码。根据需要的业务逻辑新增贴个简示（只有一个单个的imageview设置了点击事件）



```java
public class FloatingView extends BaseFloatingView implements View.OnClickListener{  
    private Context mContext;  
    private ImageView img;  
    private View root;  
  
    public FloatingView(Context context) {  
        super(context);  
        this.mContext = context;  
        initView(context);  
    }  
  
    private void initView(Context context){  
        this.mContext = context;  
        root = View.inflate(context, R.layout.layout_custom_view,null);  
  
        img = (ImageView) root.findViewById(R.id.iv_img);  
        img.setOnClickListener(this);  
    }  
  
    public static int dip2px(Context context, float dipValue) {  
  
        final float scale = context.getResources().getDisplayMetrics().density;  
  
        return (int) (dipValue * scale + 0.5f);  
    }  
  
    public boolean show(){  
        WindowManager.LayoutParams lp = new WindowManager.LayoutParams();  
        lp.width = dip2px(mContext, 160);  
        lp.height = dip2px(mContext, 90);  
  
        if (root.getParent() == null)  
            addView(root, lp);  
        super.showView(this);  
        return true;  
    }  
  
    public void dismiss() {  
        super.hideView();  
    }  
  
    @Override  
    public boolean onSingleTapUp(MotionEvent e) {  
        dismiss();  
        return true;  
    }  
  
    @Override  
    public void onClick(View view) {  
        if (view == img){  
            Toast.makeText(mContext, "imgview click!", Toast.LENGTH_SHORT).show();  
        }  
    }  
}  
```



最后activity中使用

```java
private void initFloatingView() {  
    mFloatingView = new FloatingView(this);  
    mFloatingView.show();  
}
```



[源代码地址](https://github.com/ssccbb/FloatingView)

效果演示

![20170602194053178](https://github.com/ssccbb/ssccbb.github.io/blob/master/assets/cover/20170602194053178.gif?raw=true)