---
layout: post
title: 你好, View Binding! 再次再见, findViewById!
summary: View Binding VS findViewById VS Butter Knife(Kotter Knife) VS Kotlin Android Extensions VS Data Binding
featured-img: hello-view-binding
categories: [Android]
---


# 你好, View Binding! 再次再见, findViewById!


作为一个 Android 开发者, 你一定听说过 `findViewById`. 这个方法可以根据 ID 去匹配对应的 View. 实现了类似功能或者增强了其功能的还有:

1. Butter Knife(Kotter Knife)
2. Kotlin Android Extensions
3. Data Binding
4. View Binding

## 为什么不是 findViewById/Butter Knife(Kotter Knife)/Kotlin Android Extensions ?
```kotlin
// findViewById
val fab = findViewById<FloatingActionButton>(R.id.fab)
val toolbar = findViewById<Toolbar>(R.id.toolbar)

setSupportActionBar(toolbar)
fab.setOnClickListener { view ->

}

// Kotter Knife
val fab: FloatingActionButton by bindView(R.id.fab)
val toolbar: Toolbar by bindView(R.id.toolbar)

setSupportActionBar(toolbar)
fab.setOnClickListener { view ->

}

// Kotlin Android Extensions
import kotlinx.android.synthetic.main.activity_main.*

setSupportActionBar(toolbar)
fab.setOnClickListener { view ->

}

// Data Binding & View Binding
val binding = ActivityMainBinding.inflate(layoutInflater)

setSupportActionBar(binding.toolbar)
binding.fab.setOnClickListener { view ->

}
```

### 优雅程度
可以确定的是 findViewById 和 Kotter Knife 是最不优雅的. 每初始化一个 view 都需要调用一次 `findViewById` 或 `bindView` 方法, 导致 activity 或 fragment 中充斥着许多模版代码. Kotlin Android Extensions(view cache map), Data Binding 与 View Binding (binding class)则通过生成一个中间变量的方式减少了模版代码的产生, 想象一下如果你有 20 个 view 需要初始化.

### 类型安全
```java
// API 26 之前
public final View findViewById(int id) {
    if (id < 0) {
        return null;
    }
    return findViewTraversal(id);
}

protected View findViewTraversal(int id) {
    if (id == mID) {
        return this;
    }
    return null;
}

// API 26 及以后
@Nullable
public final <T extends View> T findViewById(@IdRes int id) {
    if (id == NO_ID) {
        return null;
    }
    return findViewTraversal(id);
}

protected <T extends View> T findViewTraversal(@IdRes int id) {
    if (id == mID) {
        return (T) this;
    }
    return null;
}
```

在 API 26 之前, 初始化 view需要进行强转, 比如 `val fab = findViewById(R.id.fab) as FloatingActionButton`. 众所周知(并且我相信你也遇到过), 强转是有可能产生 `ClassCastException` 的. 尽管 Google 在 API 26 更新 `findViewById` 方法为泛型实现, 但是仍然存在强转的问题, 想象一下这段代码:  `val fab = findViewById<Toolbar>(R.id.fab)`, 在编译期不会有任何的问题, 但是很明显运行时会出错. Butter Knife 和 Kotter Knife 同样存在这样的问题.

### 空安全
首先说说 Kotlin Android Extensions 存在的问题.
```kotlin
import kotlinx.android.synthetic.main.fragment_main.*

class MainFragment : Fragment() {

    override fun onCreateView(inflater: LayoutInflater, container: ViewGroup?, savedInstanceState: Bundle?): View? {
        val rootView = inflater.inflate(R.layout.fragment_main, container, false)

        text.setOnClickListener {

        }

        return rootView
    }

}
```

```xml
<!-- fragment_main.xml -->
<androidx.constraintlayout.widget.ConstraintLayout ...>

    <androidx.appcompat.widget.AppCompatTextView
        android:id="@+id/text"
        ... />

</androidx.constraintlayout.widget.ConstraintLayout>
```

看起来没有问题对吧, 实际上运行之后:

```shell
Caused by: android.view.InflateException: Binary XML file line #24 in io.tonnyl.demo:layout/activity_main: Binary XML file line #8 in io.tonnyl.demo:layout/content_main: Error inflating class fragment
Caused by: android.view.InflateException: Binary XML file line #8 in io.tonnyl.demo:layout/content_main: Error inflating class fragment
Caused by: java.lang.NullPointerException: Attempt to invoke virtual method 'void androidx.appcompat.widget.AppCompatTextView.setOnClickListener(android.view.View$OnClickListener)' on a null object reference
```

让我们看看生成的代码:

```java
public final class MainFragment extends Fragment {
    private HashMap _$_findViewCache;

    @Nullable
    public View onCreateView(@NotNull LayoutInflater inflater, @Nullable ViewGroup container, @Nullable Bundle savedInstanceState) {
       Intrinsics.checkParameterIsNotNull(inflater, "inflater");
       View rootView = inflater.inflate(1300003, container, false);
       ((AppCompatTextView)this._$_findCachedViewById(id.text)).setOnClickListener((OnClickListener)null.INSTANCE);
       return rootView;
    }

    public View _$_findCachedViewById(int var1) {
       if (this._$_findViewCache == null) {
          this._$_findViewCache = new HashMap();
       }

       View var2 = (View)this._$_findViewCache.get(var1);
       if (var2 == null) {
          View var10000 = this.getView();
          if (var10000 == null) {
             return null;
          }

          var2 = var10000.findViewById(var1);
          this._$_findViewCache.put(var1, var2);
       }

       return var2;
    }
}
```

问题就出在 `_$_findCachedViewById` 方法 `this.getView()` 这一行, 调用时 `onCreateView()` 方法还没有返回值, 所以 `this.getView()` 返回 null, 在 `onCreateView` 中调用 `text.setOnClickListener {}` 不会有任何的错误提示, 因为 `text` 在这里会被认为是非空的.

当然, 上面的问题还是可以被解决的.

```kotlin
import kotlinx.android.synthetic.main.fragment_main.view.*

class MainFragment : Fragment() {

    override fun onCreateView(inflater: LayoutInflater, container: ViewGroup?, savedInstanceState: Bundle?): View? {
        val rootView = inflater.inflate(R.layout.fragment_main, container, false)

        rootView.text.setOnClickListener {}

        return rootView
    }

}
```

注意导入的类的变化. 再看看生成的代码的变化:

```java
public final class MainFragment extends Fragment {

    @Nullable
    public View onCreateView(@NotNull LayoutInflater inflater, @Nullable ViewGroup container, @Nullable Bundle savedInstanceState) {
       Intrinsics.checkParameterIsNotNull(inflater, "inflater");
       View rootView = inflater.inflate(1300003, container, false);
       Intrinsics.checkExpressionValueIsNotNull(rootView, "rootView");
       ((AppCompatTextView)rootView.findViewById(id.text)).setOnClickListener((OnClickListener)null.INSTANCE);
       return rootView;
    }
}
```

这里就不再通过 `_$_findViewCache` 而是直接通过 `findViewById` 实现了.

Kotlin Android Extensions 还存在一些其他的问题, Google 内部也不再使用, 具体原因可以参考 [Why kotlinx synthetic is no longer a recommended practice](https://www.reddit.com/r/androiddev/comments/ala9p2/why_kotlinx_synthetic_is_no_longer_a_recommended/) .

再说说 findViewById 和 Butter Knife(Kotter Knife) 的问题. 考虑下面的代码:
```kotlin
val fab = findViewById<FloatingActionButton>(View.NO_ID)
val fab2: FloatingActionButton by bindView(0)
```

我们都希望在编译期就可以发现问题, 而不是运行时. 但不幸的是上面的代码正好在编译期不会有任何问题而运行时会出错.

view binding 会直接创建对 view 的引用, 所以不用担心因为无效的 view ID 而产生的空指针错误. 并且, 如果一个 view 只出现在部分配置的布局中, 那么, binding class 中包含改引用的字段会被标记为 `@Nullable`.

### 为什么不是 Data Binding ?
+ 只有布局文件的根标签是 `<layout>` 时,  Data Binding 才会生成对应的 binding class, View Binding 没有这样的要求;
+ Data Binding 会影响构建的速度. Data Binding 底层其实是通过 annotation processor 实现的, 对构建速度是有负面影响的. 而 View Binding 并不是通过 annotation processor 实现的, 因此解决了 Data Binding 的性能问题.

## 什么是 View Binding ?
> View Binding 是一项使你能更轻松地编写与视图交互的代码的功能. 在模块中启用 View Binding 后, 它会为该模块中存在的每一个 XML 文件生成一个对应的绑定类(binding class). 绑定类的实例包含了对应布局中所有具有 ID 的 view 的直接引用.
>
> 大多数情况下, View Binding 可以替换 findViewById.

来自 [Android Developers](https://developer.android.com/topic/libraries/view-binding).

View Binding 第一次出现是在 2019 年的 Google I/O 大会 [What's New in Android (Google I/O'19)](https://www.youtube.com/watch?v=td3Kd7fOROw)  演讲. 

## 如何使用 View Binding ?
### 使用要求
你至少需要使用 **Android Studio 3.6 Canary 11 及以上版本** 才可以开启 View Binding.

### 设置指南
View Binding 可以逐模块(module)开启. 比如我们的项目由 2 个模块(A 和 B)组成, 我们可以选择只在模块 A 启用 View Binding 而不会对模块 B 产生影响. 在模块中启用  View Binding, 首先需要在该模块的 `build.gradle` 文件中添加以下代码:
```gradle
android {
    ...
    viewBinding {
        enabled = true
    }
}
```

如果想要在生成 binding class 时忽略某个布局文件, 我们需要在该布局文件的根元素添加 `tools:viewBindingIgnore="true"` 属性.
```xml
<LinearLayout
        ...
        tools:viewBindingIgnore="true" >
    ...
</LinearLayout>
```

### 使用方法
和 Data Binding 一样, View Binding 会将 XML 文件的下划线风格的名称转换生成一个驼峰风格并以 `Binding` 结尾的 binding class. 

例如我们有个 `result_profile.xml` 布局文件:
```xml
<LinearLayout ... >
    <TextView android:id="@+id/name" />
    <ImageView android:cropToPadding="true" />
    <Button android:id="@+id/button"
        android:background="@drawable/rounded_button" />
</LinearLayout>
```

生成的 binding class 的名称就是 `ResultProfileBinding`. 这个类有两个字段: 一个叫 `name` 的 `TextView` 和一个叫 `button` 的 `Button`, 布局文件中的 `ImageView` 因为没有 ID, 所以 binding class 中没有对其的引用.

每一个 binding class 都包含了一个 `getRoot()` 的方法, 提供了一个相应布局文件的根 view 的直接引用. 在上面的例子中, `ResultProfileBinding` 类中的 `getRoot()` 方法返回了根 view `LinearLayout`.

我们可以调用 `inflate()` 静态方法来获取生成的 binding class 的实例. 通常来说, 你需要调用 `setContentView()` 方法, 将生成类的根 view 作为参数传递, 作为屏幕上的内容. 在上面的例子中, 我们可以在 activity 中调用 `ResultProfileBinding.inflate()`.

```kotlin
private lateinit var binding: ResultProfileBinding

@Override
fun onCreate(savedInstanceState: Bundle) {
    super.onCreate(savedInstanceState)
    binding = ResultProfileBinding.inflate(layoutInflater)
    setContentView(binding.root)
}
```

然后就可以通过 binding class 引用任何 view:
```kotlin
binding.name.text = viewModel.name
binding.button.setOnClickListener { viewModel.userClicked() }
```

## 结语
没错, 我们 **又** 要[和 findViewById 说再见](https://antonioleiva.com/kotlin-android-extensions/)了, View Binding 确实足够简单, 也足够强大. Jake Wharton 也把 [Kotter Knife](https://github.com/JakeWharton/kotterknife) 标记为了「废弃」, 并且推荐使用 View Binding.

有机会的话, 推荐你尝试一下吧.

欢迎在这些平台关注我:
+ [GitHub](https://github.com/TonnyL)
+ [知乎](https://www.zhihu.com/people/lizhaotailang/)
+ [Medium](https://medium.com/@TonnyL)
+ [微博](https://weibo.com/u/5313690193/)
+ [Instagram](https://www.instagram.com/tonny_lztl/)
