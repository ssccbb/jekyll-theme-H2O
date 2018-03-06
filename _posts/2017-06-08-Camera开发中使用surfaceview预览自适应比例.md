---
layout: post
title: 'Camera开发中使用surfaceview预览自适应比例'
date: 2017-06-08
author: sung
cover: ''
tags: Android开发 Surfaceview
---

csdn：<http://blog.csdn.net/sung_ll>



### 前言

app需要增加类似美拍的段视频拍摄功能，在实际的开发中碰到surface view预览的变形。参考了网上的资料，提出了解决办法



### 思路

在camera的开发中，因为不同手机硬件的原因，所支持的分辨率都不一样，所以在surface view预览的时候需要同时考虑到surface view的大小比例以及camera的画面比例，然而在设置比例尺的时候如若传入不支持的宽高，camera的画面是传递不过来的，所以只能根据手机支持的比例来改变surface大小及画面。注，surfaceveiw比例、camera画面比例、pictrue size比例三者必须一致。



### 正文

##### 本文只讲问题的处理，camera开发不作赘述

首先是找到关键的设置，有两个地方

1.是surface view初始化时大小的设置

```java
mSurfaceView.setLayoutParams(lp); 
```

2.是视频比例的设置

```java
mParameters.setPreviewSize(previewSize.width,previewSize.height);  
```



接下来分析

因为分辨率必须是手机硬件支持的尺寸才能显示出画面，所以预览的设置只有以下两种情况：

1固定比例，surface大小根据 屏幕宽 和 比例尺 动态设置高度，camera分辨率传入固定值

市场安卓机器一般支持的尺寸高度有：240，480，720，1080等。屏幕比例大多4:3或者16:9，使用时根据需要自定，具体值可用以下api获取

```java
mParameters.getSupportedPreviewFrameRates();  
```

使用固定比例时，机器的不同屏幕比例也不同，当固定比例不同等于屏幕时就会出现预览图像上下白边的情况，所以这时候就需要使用第二种

2动态选择比例

初始化时获取屏幕的比例

```java
	/** 获取屏幕比例*/  
    public static float getScreenScale(Context context){  
        Display display = ((WindowManager) context.getSystemService(Context.WINDOW_SERVICE)).getDefaultDisplay();  
        return (float) (display.getHeight()) / (float) (display.getWidth());  
    }  
```

获取屏幕比例height和width计算时需要时float或者double类型，结果带小数



传入获取到的支持的分辨率列表list，最小宽度，屏幕比例。返回合适的camera分辨率

```java
	public Camera.Size getPreviewSize(List<Camera.Size> list, int th, float scale){  
        Collections.sort(list, sizeComparator);  
  
        //Log.e(tag,"prepareCameraParaments:scale-"+scale);  
        int i = 0;  
        for(Camera.Size s:list){  
            if((s.width > th) && equalRate(s, scale)){  
                Log.i(tag, "最终设置预览尺寸:w = " + s.width + "h = " + s.height);  
                break;  
            }  
            i++;  
        }  
  
        return list.get(i);  
    } 
```

最后在同时修改surfaceview满屏和设置camera分辨率



### 延伸

贴上尺寸获取的工具类

```java
/** 
 * Created by sung on 2017/6/8. 
 */  
  
public class CameraSizeUtils {  
    private static final String tag = "yan";  
    private CameraSizeComparator sizeComparator = new CameraSizeComparator();  
    private static CameraSizeUtils myCamPara = null;  
    public static float SCREEN_SCALE = 4/3;  
  
    private CameraSizeUtils(){  
  
    }  
    public static CameraSizeUtils getInstance(){  
        if(myCamPara == null){  
            myCamPara = new CameraSizeUtils();  
            return myCamPara;  
        }  
        else{  
            return myCamPara;  
        }  
    }  
  
    public Camera.Size getPreviewSize(List<Camera.Size> list, int th, float scale){  
        Collections.sort(list, sizeComparator);  
  
        //Log.e(tag,"prepareCameraParaments:scale-"+scale);  
        int i = 0;  
        for(Camera.Size s:list){  
            if((s.width > th) && equalRate(s, scale)){  
                Log.i(tag, "最终设置预览尺寸:w = " + s.width + "h = " + s.height);  
                break;  
            }  
            i++;  
        }  
  
        return list.get(i);  
    }  
  
    public Camera.Size getPictureSize(List<Camera.Size> list, int th, float scale){  
        Collections.sort(list, sizeComparator);  
  
        int i = 0;  
        for(Camera.Size s:list){  
            if((s.width > th) && equalRate(s, scale)){  
                Log.i(tag, "最终设置图片尺寸:w = " + s.width + "h = " + s.height);  
                break;  
            }  
            i++;  
        }  
  
        return list.get(i);  
    }  
  
    public boolean equalRate(Camera.Size s, float rate){  
        float r = (float)(s.width)/(float)(s.height);  
        if(Math.abs(r - rate) <= 0.2)  
        {  
            return true;  
        }  
        else{  
            return false;  
        }  
    }  
  
    public  class CameraSizeComparator implements Comparator<Camera.Size> {  
        //按升序排列    
        public int compare(Camera.Size lhs, Camera.Size rhs) {  
            // TODO Auto-generated method stub    
            if(lhs.width == rhs.width){  
                return 0;  
            }  
            else if(lhs.width > rhs.width){  
                return 1;  
            }  
            else{  
                return -1;  
            }  
        }  
  
    }  
}  
```

