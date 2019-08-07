---
layout: post
title: "Android沉浸状态栏"
date: 2019-08-07
description: "Android沉浸状态栏实现方式"
tag: android
---


沉浸式状态栏的实现方式有两种

<h3> 第一种方法：代码中设置</h3>

   

```java
 if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.KITKAT) {
    	int flagTranslucentStatus = WindowManager.LayoutParams.FLAG_TRANSLUCENT_STATU;
    	int flagTranslucentNavigation = 	
            					WindowManager.LayoutParams.FLAG_TRANSLUCENT_NAVIGATIO;
    	if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
            //5.x开始需要把颜色设置透明，否则导航栏会呈现系统默认的浅灰色
            Window window = activity.getWindow();
            View decorView = window.getDecorView();
            //两个 flag 要结合使用，表示让应用的主体内容占用系统状态栏的空间
            int option = View.SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN
                    | View.SYSTEM_UI_FLAG_LAYOUT_STABLE;
            decorView.setSystemUiVisibility(option);
            					  		
           window.addFlags(WindowManager.LayoutParams.FLAG_DRAWS_SYSTEM_BAR_BACKGROUNDS);
            window.setStatusBarColor(Color.TRANSPARENT);
            //导航栏颜色也可以正常设置
			//  window.setNavigationBarColor(Color.TRANSPARENT);
        } else {
            Window window = activity.getWindow();
            WindowManager.LayoutParams attributes = window.getAttributes();
            int flagTranslucentStatus =                         															WindowManager.LayoutParams.FLAG_TRANSLUCENT_STATUS;
            
            int flagTranslucentNavigation = 
                				  WindowManager.LayoutParams.FLAG_TRANSLUCENT_NAVIGATION;
            attributes.flags |= flagTranslucentStatus;
			// attributes.flags |= flagTranslucentNavigation;
            window.setAttributes(attributes);
        }
		ViewGroup contentFrameLayout = (ViewGroup)findViewById(android.R.id.content);
		View parentView = contentFrameLayout.getChildAt(0);
        parentView.setFitsSystemWindows(value);
}
```
<h3> 第二种方法：通过设置 Theme 主题设置状态栏透明</h3>

API21 之后（也就是 android 5.0 之后）的状态栏，会默认覆盖一层半透明遮罩。且为了保持4.4以前系统正常使用，故需要三份 style 文件，即默认的values（不设置状态栏透明）、values-v19、values-v21（解决半透明遮罩问题）。
    
```xml
//valuse
<style name="TranslucentTheme" parent="AppTheme">
</style>

// values-v19。v19 开始有 android:windowTranslucentStatus 这个属性
<style name="TranslucentTheme" parent="Theme.AppCompat.Light.NoActionBar">
	<item name="android:windowTranslucentStatus">true</item>
	<item name="android:windowTranslucentNavigation">true</item>
</style>

// values-v21。5.0 以上提供了 setStatusBarColor()  方法设置状态栏颜色。
<style name="TranslucentTheme" parent="Theme.AppCompat.Light.NoActionBar">
	<item name="android:windowTranslucentStatus">false</item>
	<item name="android:windowTranslucentNavigation">true</item>
	<!--Android 5.x开始需要把颜色设置透明，否则导航栏会呈现系统默认的浅灰色-->
	<item name="android:statusBarColor">@android:color/transparent</item>
</style>
```
这种实现方式，底部导航栏的虚拟按键覆盖住了布局；

从 style 中的属性我们猜测，我们需要将 `<item name="android:windowTranslucentNavigation">false</item>`  属性设置为 false ，是 Navigation Bar 不透明，但效果却并没有变成我们想象的那样，`Status Bar` 的样式完全失效。

事实上，`android:windowTranslucentNavigation`  这个属性额外设置了 `SYSTEM_UI_FLAG_LAYOUT_STABLE`  和 `SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN` 两个 flag。所以我们需要再绘制完 Status Bar 以后重新请求这两个 flag 。

在 onCrate 方法中调用

```java
getWindow().getDecorView().setSystemUiVisibility(View.SYSTEM_UI_FLAG_LAYOUT_STABLE | View.SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN);
```