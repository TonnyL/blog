---
layout: post # needs to be post
title: 简洁优雅地实现夜间模式
summary: 本次教程将教你如何简洁优雅地实现夜间模式
featured-img: design
categories: [Android]
---

2019.04.07 更新.

## 前言
Android 6.0 Marshmallow 预览版中曾经短暂出现过相关的夜间模式的功能，只是在正式版中被移除了，在 Android 7.0 Nougat 上，用户们再次经历了「得而复失」的遗憾，在开发者预览版中，夜间模式和暗色模式先是开启，然后有再次被移除。而在正式版中，夜间模式也没有出现。但其实相关的代码一直存在于系统中，只是默认没有被开启。如何开启这项功能，可以参考少数派的这一篇文章，[帮你找回 Android 7.0 夜间模式的 2 款应用](http://sspai.com/35273)。

不过，今天要介绍的主要内容并不是关于系统的夜间模式，而是如何给我们开发的 APP 添加夜间模式的功能。毫不夸张的说，夜间模式现在已经是阅读类 App 的标配了。事实上，日间模式与夜间模式就是给 APP 定义并应用两套不同颜色的主题。用户可以自动或者手动的开启。我们先看两个我认为实现地很优雅的例子：知乎和 Twitter 。

![Night Mode of Zhihu&Twitter.gif](https://i.loli.net/2018/03/27/5ab9e2289ecb6.gif)

这两个 APP 在切换的工程中，并没有出现闪现黑屏的情况，切换也比较顺滑。我们的目标就是利用 Support Library 实现同样的效果。

## 实现
### 添加依赖
```gradle
implementation “androidx.appcompat:appcompat:1.1.0-alpha04”
```

### 准备资源
1. 让我们自己的主题继承并应用DayNight主题:
	
	```xml
	<style name="AppTheme" parent="Theme.AppCompat.DayNight">
	        <item name="colorPrimary">@color/colorPrimary</item>
	        <item name="colorPrimaryDark">@color/colorPrimaryDark</item>
	        <item name="colorAccent">@color/colorAccent</item>
	        <!--customize your theme here-->
	</style>
	```

	> 如果你使用了 Material Design Components, 那么从 v1.1.0 release 开始, 你也可以使用 `Theme.MaterialComponents.DayNight` , 本文的其他内容保持一致. 

2. 新建夜间模式资源文件夹: 

	在 `res` 目录下新建 `values-night` 文件夹，然后在此目录下新建 `colors.xml` 文件在夜间模式下的应用的资源。当然也可以根据需要新建 `drawable-night`, `layout-night` 等后缀为 `-night` 的夜间资源文件夹。
我的 `values` 和 `values-night` 目录下的 `colors.xml` 的内容如下:

	```xml
	<?xml version="1.0" encoding="utf-8"?>
	<!--values-colors.xml-->
	<resources>
	    <color name="colorPrimary">#009688</color>
	    <color name="colorPrimaryDark">#00796B</color>
	    <color name="colorAccent">#009688</color>
	    <color name="textColorPrimary">#616161</color>
	    <color name="viewBackground">@android:color/white</color>
	</resources>
	<!--values-night-colors.xml-->
	<?xml version="1.0" encoding="utf-8"?>
	<resources>
	    <color name="colorPrimary">#35464e</color>
	    <color name="colorPrimaryDark">#212a2f</color>
	    <color name="colorAccent">#212a2f</color>
	    <color name="textColorPrimary">#616161</color>
	    <color name="viewBackground">#212a2f</color>
	</resources>
	```
	
3. 使Activity继承自AppCompatActivity:

	```java
	public class MainActivity extends AppCompatActivity {
		// code here
		
	@Override
	    protected void onCreate(Bundle savedInstanceState) {
	    }
	}
	```
	
### 应用
#### 静态应用
在Application的继承类下设置初始主题:

```java
public class App extends Application {
    
	@Override
    public void onCreate() {
        super.onCreate();
		AppCompatDelegate.setDefaultNightMode(AppCompatDelegate.MODE_NIGHT_YES);
		// other code here
	}
}
```

这里 `AppCompatDelegate.setDefaultNightMode()` 方法可以接受的参数值有 4 个：

+ [MODE_NIGHT_NO](https://developer.android.com/reference/androidx/appcompat/app/AppCompatDelegate.html#MODE_NIGHT_NO). 一直应用日间(light)主题.
+ [MODE_NIGHT_YES](https://developer.android.com/reference/androidx/appcompat/app/AppCompatDelegate.html#MODE_NIGHT_YES). 一直使用夜间(dark)主题.
+ [MODE_NIGHT_FOLLOW_SYSTEM](https://developer.android.com/reference/androidx/appcompat/app/AppCompatDelegate.html#MODE_NIGHT_FOLLOW_SYSTEM)(默认). 跟随系统设置, 在 Android Pie 及以上日/夜间主题是系统设置项.
+ MODE_NIGHT_AUTO_BATTERY. 当设备开启了“省电模式”时, 更改为夜间模式. 反之则更改为日间模式.(这是在 v1.1.0-alpha03 新增的选项)
+ [MODE_NIGHT_AUTO](https://developer.android.com/reference/androidx/appcompat/app/AppCompatDelegate.html#MODE_NIGHT_AUTO) 和 MODE_NIGHT_AUTO_TIME. 根据当前时间在 day/night 主题间切换. (在 v1.1.0-alpha03 被废弃)

我们可以在任何时候调用这个方法，因为这个方法是静态的。但是这个值并不是一直存在的，每次在开启进程时需要重新设置。在上面的代码中，我是在 `onCreate()` 方法中设置的，网上也有大神建议在 Activity 或者 Application 的 static 代码块中设置。如下所示：

```java
public class App extends Application {
		static {
		    AppCompatDelegate.setDefaultNightMode(AppCompatDelegate.MODE_NIGHT_YES);
		}
	    
		@Override
	    public void onCreate() {
	        super.onCreate();
			// other code here
		}
}
```

#### setLocalNightMode()
你也可以通过调用 AppCompatDelegate’s `setLocalNightMode()` 方法, 复写每个组件的默认值. 当你确认只有一部分组件应该使用日/夜间模式功能, 或对于开发而言, 不用等到天黑你就可以测试你所写的布局, 十分方便.

如果你调用了这个方法, 那改变主题是必须的, 通过调用 [recreate()](https://developer.android.com/reference/android/app/Activity.html#recreate%28%29) 方法, 自动地重建 Activity, 这样新的主题才会被应用上. 这也是测试你的 Activity + Fragments 是否正确保存了它们的 [instance state](https://developer.android.com/reference/android/app/Activity#onSaveInstanceState%28android.os.Bundle%29) 的好机会.

#### 动态应用
虽然上面的静态应用的设置非常简单，但是这种方式的应用场景还是太少了。我们更多的还是需要动态的根据需要动态的切换。

1. 检测当前主题模式
	
	```java
	int mode = getResources().getConfiguration().uiMode & Configuration.UI_MODE_NIGHT_MASK;
	```
2. 设置主题
	
	```java
	if(mode == Configuration.UI_MODE_NIGHT_YES) {
		AppCompatDelegate.setDefaultNightMode(AppCompatDelegate.MODE_NIGHT_NO);
	} else if(mode == Configuration.UI_MODE_NIGHT_NO) {
		AppCompatDelegate.setDefaultNightMode(AppCompatDelegate.MODE_NIGHT_YES);
	} else {
		// blah blah
	}
	recreate();
	```
	
在调用 `recreate()` 方法之前，还可以创建一些动画进行过渡。而且，众所周知，调用 `recreate()` 需要对一些数据进行保存，例如 fragment，CheckBox, RadioBox 等。如下所示:

```java
public class MainFragment extends Fragment {
	
	@Override
   	public void onCreate(@Nullable Bundle savedInstanceState) {
       	super.onCreate(savedInstanceState);
   	 	if (savedInstanceState != null) {
   	 		FragmentManager manager = getChildFragmentManager();
			doubanMomentFragment = (DoubanMomentFragment) manager.getFragment(savedInstanceState, "douban");
   	 	} else {
   	 		doubanMomentFragment = DoubanMomentFragment.newInstance();
   	 	}
   	}
   	
	@Override
   	public void onSaveInstanceState(Bundle outState) {
      	 super.onSaveInstanceState(outState);
      	 FragmentManager manager = getChildFragmentManager();
      	 manager.putFragment(outState, "douban", doubanMomentFragment);
}
```

我们也可以把主题的值存储到SharedPreference中，已便于应用在下一次启动时自动应用主题。

## Q&A
+ Q: 系统默认的颜色不合我的口味怎么办？
  
	A: 使用主题属性,例如: `textColor:?android:attr/textColorPrimary`, `color:?attr/colorControlNormal` 等。
  
+ Q: 为什么我的 WebView 颜色没有变化？

	A: 因为 WebView 不能使用主题属性。WebView 的颜色实际上取决于网页内容颜色。网页的默认背景色是白色，所以尽管设置了主题为 `AppCompatDelegate.MODE_NIGHT_YES`，网页仍然是白色，所以看起来就很不搭了。所以，网页的内容和背景色等资源也需要适配了。

+ Q: 关于系统夜间模式:
	A: Android Pie 有一个系统的夜间模式, 可以在 "[开发者选项](https://developer.android.com/studio/debug/dev-options)" 中开启. 在 Android Pie 及以上的设备上, 我们应该默认使用 `MODE_NIGHT_FOLLOW_SYSTEM`, 这样用户设备的设置才能正常运心. 
	
## 应用内设置
推荐为用户提供默认主题的设置项. 推荐选项和文本内容:

+ "日间"(MODE_NIGHT_NO)
+ “夜间”(MODE_NIGHT_YES)
+ “通过省电模式设置”(MODE_NIGHT_AUTO_BATTERY)
+ “使用系统默认”(MODE_NIGHT_FOLLOW_SYSTEM). 仅在 API 28+ 显示. 显示时应该为默认选中.

常用的方法为通过 [ListPreference](https://developer.android.com/guide/topics/ui/settings/components-and-attributes) 实现.

## 更新主题和样式
除了调用 AppCompat 之外, 你还需要做一些别的工作来更新你的主题, 样式和布局, 这样它们才能无缝衔接日间和夜间模式. 

粗略估计一下，需要做的事情就是当你可以使用主题属性时一定要使用.最需要了解的是:

+ `?android:attr/textColorPrimary`. 常规文字颜色. 日间模式下接近黑色, 夜间模式下接近白色. 包含了不可用状态(disabled state). 
+ `?attr/colorControlNormal`. 常规图标颜色. 包含了不可用状态(disabled state). 

使用 Material Design Components 能让问题变的简单, 它的属性(像 `?attr/colorSurface` 和 `?attr/colorOnSurface`) 提供了容易生成的主题颜色以供使用. 当然这些属性你也可以在自己的主题中定制.
	
## 代码
本项目代码 [PaperPlane](https://github.com/TonnyL/PaperPlane) . 运行效果:

![PaperPlane](https://i.loli.net/2018/03/27/5ab9e550323ff.gif)
