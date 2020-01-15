---
layout: post
title: 将 Android 项目迁移至 Kotlin Coroutines
summary: 本文将介绍如何将一个 Android 项目迁移至 Kotlin Coroutines
featured-img: shane-rounce-205187
categories: [Android, Kotlin]
---

## 前言
在2017年 Google I/O 大会上, Google 宣布了 Android 平台对 Kotlin 语言的官方支持. 我思考了很久如何向没有听说过 Kotlin 语言的开发者介绍它呢? 用这个[知乎的段子](https://www.zhihu.com/question/48633827/answer/254037101)应该是最合适不过了:

> Scala：想解决 Java 表达能力不足的问题
>
> Groovy：想解决 Java 语法过于冗长的问题
>
> Clojure：想解决 Java 没有函数式编程的问题
>
> Kotlin：想解决 Java

段子归段子, 事实上, Kotlin 在国外公司的应用已经十分广泛, 如 [Pinterest](https://www.youtube.com/watch?v=xRDqDe4rxkM), [Gradle](https://blog.gradle.org/kotlin-meets-gradle), [Evernote](https://blog.evernote.com/tech/2017/01/26/android-state-library/), [Uber](https://www.reddit.com/r/androiddev/comments/5sihp0/2017_whos_using_kotlin/ddfmkf7/), [Trello](https://github.com/trello/RxLifecycle/tree/1.x/rxlifecycle-kotlin), [Square](https://github.com/square/moshi/tree/master/kotlin), [Google](https://android.googlesource.com/platform/frameworks/data-binding/+/master/compiler/src/main/kotlin/android/databinding/tool/writer/) 等等. 那么为什么要使用 Kotlin 呢?换言之, 相比于 Java, Kotlin 能给我带来什么好处?

- [Lambda 表达式](https://www.kotlincn.net/docs/reference/lambdas.html) + [内联函数](https://www.kotlincn.net/docs/reference/inline-functions.html) = 高性能自定义控制结构
- [扩展函数](https://www.kotlincn.net/docs/reference/extensions.html)
- [空安全](https://www.kotlincn.net/docs/reference/null-safety.html)
- [智能类型转换](https://www.kotlincn.net/docs/reference/typecasts.html)
- [字符串模板](https://www.kotlincn.net/docs/reference/basic-types.html#%E5%AD%97%E7%AC%A6%E4%B8%B2)
- [属性](https://www.kotlincn.net/docs/reference/properties.html)
- [主构造函数](https://www.kotlincn.net/docs/reference/classes.html)
- [一等公民的委托](https://www.kotlincn.net/docs/reference/delegation.html)
- [变量和属性类型的类型推断](https://www.kotlincn.net/docs/reference/basic-types.html)
- [单例](https://www.kotlincn.net/docs/reference/object-declarations.html)
- [声明处型变 & 类型投影](https://www.kotlincn.net/docs/reference/generics.html)
- [区间表达式](https://www.kotlincn.net/docs/reference/ranges.html)
- [操作符重载](https://www.kotlincn.net/docs/reference/operator-overloading.html)
- [伴生对象](https://www.kotlincn.net/docs/reference/classes.html#%E4%BC%B4%E7%94%9F%E5%AF%B9%E8%B1%A1)
- [数据类](https://www.kotlincn.net/docs/reference/data-classes.html)
- [分离用于只读和可变集合的接口](https://www.kotlincn.net/docs/reference/collections.html)
- [协程](https://www.kotlincn.net/docs/reference/coroutines.html)
- …<sup>[[1]](#reference-1)</sup>

解决了 Why 的问题, 下面我们来解决 How to 的问题.

## 开始迁移

首先介绍一下本次项目的相关内容. 项目名称为[纸飞机](https://github.com/TonnyL/PaperPlane)(https://github.com/TonnyL/PaperPlane), 是一个集合知乎日报, 豆瓣一刻和果壳精选的综合性阅读 App<sup>[[2]](#reference-2)</sup>, 项目参考了 Google 推出的 [Android Architecture Blueprints](https://github.com/googlesamples/android-architecture) 中 [todo-mvp](https://github.com/googlesamples/android-architecture/tree/todo-mvp/) 的 MVP 架构, 本次迁移仍然沿袭 MVP 架构, 主要的变化来自 Kotlin 语言以及 [Kotlin Coroutines](https://github.com/Kotlin/kotlinx.coroutines/) 的应用.

![Screenshot of PaperPlane](https://i.loli.net/2020/01/05/1ET8SMdAiCnIokj.png "Screenshot of PaperPlane")

### Kotlin IDE 插件
如果你在使用 Android Studio 2.3 及以下版本, 请升级到3.0及以上吧, 如果你还在使用 Eclipse 开发 Android 项目, 嗯…🤔⬇️

![If you still develop Android on Eclipse](https://i.loli.net/2020/01/05/ykwgS4HJbWPqLYv.jpg "If you still develop Android on Eclipse")

Android Studio 3.0 集成了 Kotlin IDE 插件, 你可以在 Android Studio -> Preferences -> Plugins -> Kotlin 找到. 当然, 你要是不嫌烦的话, 可以卸载后重新安装,  安装完成后重启 Android Studio(我之前遇到过升级 Kotlin 版本后项目炸裂的情况, 使用此方法有奇效). 

### 升级 Gradle
在项目的 [build.gradle](https://github.com/TonnyL/PaperPlane/blob/master/build.gradle) 文件中添加对 Kotlin 的支持:

```gradle
buildscript {
    ext.versions = [
            'android_gradle'            : '3.2.0-alpha15',
            'kotlin'                    : '1.2.41',
            'support_library'           : '27.1.1',
            'arch_room'                 : '1.1.0',
            'retrofit'                  : '2.4.0',
            'okhttp_logging_interceptor': '3.10.0',
            'date_time_picker'          : '3.6.0',
            'coroutines'                : '0.22.5',
            'mockito'                   : '2.8.47',
            'hamcrest'                  : '1.3',
            'glide'                     : '4.7.1',
            'junit'                     : '4.12',
            'support_test'              : '1.0.1',
            'espresso'                  : '3.0.1'
    ]
    repositories {
        google()
        jcenter()
    }
    dependencies {
        classpath "com.android.tools.build:gradle:${versions.android_gradle}"
        // Add the classpath to support Kotlin
        classpath "org.jetbrains.kotlin:kotlin-gradle-plugin:${versions.kotlin}"
    }
}
```

仅仅添加 classpath 还不够, 还需要在 Module 级别(如 app 目录下)的 [build.gradle](https://github.com/TonnyL/PaperPlane/blob/master/app/build.gradle) 添加 Kotlin 标准库依赖和 Kotlin 相关插件等:

```gradle
apply plugin: 'com.android.application'
apply plugin: 'kotlin-android'
apply plugin: 'kotlin-android-extensions'
apply plugin: 'kotlin-kapt'

android {
    // ...
    
    kapt {
        correctErrorTypes = true
    }
}

// enable Parcelize
androidExtensions {
    experimental = true
}

// enable coroutines
kotlin {
    experimental {
        coroutines "enable"
    }
}

dependencies {
    implementation fileTree(include: ['*.jar'], dir: 'libs')
	
	// ...

	// Kotlin standard library
	implementation "org.jetbrains.kotlin:kotlin-stdlib-jdk7:${versions.kotlin}"

	implementation "com.github.bumptech.glide:glide:${versions.glide}"
	// use kapt instead of annotationProcessor
	kapt "com.github.bumptech.glide:compiler:${versions.glide}"

	// Kotlin coroutines
	implementation "org.jetbrains.kotlinx:kotlinx-coroutines-core:${versions.coroutines}"
	implementation "org.jetbrains.kotlinx:kotlinx-coroutines-android:${versions.coroutines}"

	// ...
}
```

使用 Kotlin 后, `annotationProcessor` 就不要再用了, 取而代之的是 `kapt`:

```gradle
apply plugin: 'kotlin-kapt'

kapt "com.github.bumptech.glide:compiler:${versions.glide}"
```

为了使用 Coroutines, 我们还需要开启 kotlin-experimental, 这是一项处于实验中的特性.

```gradle
kotlin {
    experimental {
        coroutines "enable"
    }
}
```

到这里, Kotlin 的开发环境就配置好了. 以上是 Android Studio 的配置教程, 据我所知使用 Intellij IDEA 进行 Android 开发的开发者也有很多, 配置方式其实大同小异, 毕竟 Android Studio 也是 Powered by Intellij IDEA. 

### 将 Java 代码转换为 Kotlin代码

Android Studio 集成了一个将 Java 代码转换为 Kotlin 代码的工具, 选择 Code -> Convert Java File to Kotlin File (快捷键 <kbd>Option ⌥</kbd> + <kbd>Shift ⇧</kbd> + <kbd>Command ⌘</kbd> + <kbd>K</kbd> )即可使用, 非常简单. 

转换后的代码可能还需要我们手动的修改. 我们来看个实例, 我们将一个名为 `ZhihuDailyNewsQuestion` 的 POJO 类由 [Java](https://github.com/TonnyL/PaperPlane/commit/45476a3ce62190e65cedceea4d4ae188f6b87e84#diff-f4884ff97bce517c53a617c4ef8ead88) 转换为 [Kotlin](https://github.com/TonnyL/PaperPlane/commit/45476a3ce62190e65cedceea4d4ae188f6b87e84#diff-af319705a83bfaf50fae7b8b7a793134):

[ZhihuDailyNewsQuestion.java](https://github.com/TonnyL/PaperPlane/commit/45476a3ce62190e65cedceea4d4ae188f6b87e84#diff-f4884ff97bce517c53a617c4ef8ead88)

```java
/**
 * Immutable model class for zhihu daily news question.
 * Entity class for {@link com.google.gson.Gson} and {@link android.arch.persistence.room.Room}.
 */
@Entity(tableName = "zhihu_daily_news")
@TypeConverters(StringTypeConverter.class)
public class ZhihuDailyNewsQuestion {

    @ColumnInfo(name = "images")
    @Expose
    @SerializedName("images")
    private List<String> images;

    @ColumnInfo(name = "type")
    @Expose
    @SerializedName("type")
    private int type;

    @PrimaryKey
    @ColumnInfo(name = "id")
    @Expose
    @SerializedName("id")
    private int id;

    @ColumnInfo(name = "ga_prefix")
    @Expose
    @SerializedName("ga_prefix")
    private String gaPrefix;

    @ColumnInfo(name = "title")
    @Expose
    @SerializedName("title")
    private String title;

    @ColumnInfo(name = "favorite")
    @Expose
    private boolean favorite;

    @ColumnInfo(name = "timestamp")
    @Expose
    private long timestamp;

    public List<String> getImages() {
        return images;
    }

    public void setImages(List<String> images) {
        this.images = images;
    }

    public int getType() {
        return type;
    }

    public void setType(int type) {
        this.type = type;
    }

    public int getId() {
        return id;
    }

    public void setId(int id) {
        this.id = id;
    }

    public String getGaPrefix() {
        return gaPrefix;
    }

    public void setGaPrefix(String gaPrefix) {
        this.gaPrefix = gaPrefix;
    }

    public String getTitle() {
        return title;
    }

    public void setTitle(String title) {
        this.title = title;
    }

    public boolean isFavorite() {
        return favorite;
    }

    public void setFavorite(boolean favorite) {
        this.favorite = favorite;
    }

    public long getTimestamp() {
        return timestamp;
    }

    public void setTimestamp(long timestamp) {
        this.timestamp = timestamp;
    }
}
```



[ZhihuDailyNewsQuestion.kt](https://github.com/TonnyL/PaperPlane/commit/45476a3ce62190e65cedceea4d4ae188f6b87e84#diff-af319705a83bfaf50fae7b8b7a793134)

```kotlin
/**
 * Immutable model class for zhihu daily news question.
 * Entity class for [com.google.gson.Gson] and [android.arch.persistence.room.Room].
 */
@Entity(tableName = "zhihu_daily_news")
@TypeConverters(StringTypeConverter::class)
@Parcelize
@SuppressLint("ParcelCreator")
data class ZhihuDailyNewsQuestion(

        @ColumnInfo(name = "images")
        @Expose
        @SerializedName("images")
        val images: List<String>?,

        @ColumnInfo(name = "type")
        @Expose
        @SerializedName("type")
        val type: Int,

        @PrimaryKey
        @ColumnInfo(name = "id")
        @Expose
        @SerializedName("id")
        val id: Int = 0,

        @ColumnInfo(name = "ga_prefix")
        @Expose
        @SerializedName("ga_prefix")
        val gaPrefix: String,

        @ColumnInfo(name = "title")
        @Expose
        @SerializedName("title")
        val title: String,

        @ColumnInfo(name = "favorite")
        @Expose
        var isFavorite: Boolean = false,

        @ColumnInfo(name = "timestamp")
        @Expose
        var timestamp: Long = 0

) : Parcelable
```

我们首先从代码行就可以很清晰的看出来, Kotlin 版本(76行)相比 Java 版本(129行)减少了约40%, 而且请注意, Java 版本没有包含序列化(Parcelable)的部分. 并且, Kotlin 的 Data Class 自动生成了 `getters`、 `setters`、 `equals()`、 `hashCode()`、 `toString()` 以及 `copy()`  等等方法.

Data Class 只是 Kotlin 众多特性的一种, 如果你想要了解更多内容, 请访问 [Kotlin 官方网站](https://kotlinlang.org/docs/reference/comparison-to-java.html) 或 [Kotlin 语言中文站](https://www.kotlincn.net/). 

### 迁移至 Coroutines
本次文章的重点便是如何迁移至 Coroutines.

>  Coroutines( 协程) 通过将复杂性放入库来简化异步编程。程序的逻辑可以在协程中顺序地表达，而底层库会为我们解决其异步性。该库可以将用户代码的相关部分包装为回调、订阅相关事件、在不同线程（甚至不同机器）上调度执行，而代码则保持如同顺序执行一样简单。

官方定义总是这样, 将一堆容易理解的文字组合成难以理解的句子🙃. 简单来说, Coroutines 简化了异步编程, 让我们可以顺序地表达程序. 同时, Coroutines 提供了一种避免阻塞线程并用更廉价,更可控的操作替代线程阻塞的方法 - 协程挂起<sup>[[3]](#reference-3)</sup>. 

在未迁移之前, 我们所有的异步操作都是通过实现了 `XXXXDataSource` 接口的 `XXXXRepository`, `XXXXLocalDataSource`, `XXXXRemoteDataSource`实现的(为了方便起见, 如无特别说明, 后面均以知乎日报的实现部分举例).

`ZhihuDailyNewsDataSource`中的方法因为可能进行密集操作, 因此将这些方法添加 `suspend` 关键字. 将 callback 替换为 返回 `Result` 对象. 

[ZhihuDailyNewsDataSource.java](https://github.com/TonnyL/PaperPlane/commit/45476a3ce62190e65cedceea4d4ae188f6b87e84#diff-9aceb53bb68f7eece575a13d0f4d42c0)

```Java
public interface ZhihuDailyNewsDataSource { 

    interface LoadZhihuDailyNewsCallback {  

        void onNewsLoaded(@NonNull List<ZhihuDailyNewsQuestion> list);  

        void onDataNotAvailable();  

    }   

    interface GetNewsItemCallback { 

        void onItemLoaded(@NonNull ZhihuDailyNewsQuestion item);    

        void onDataNotAvailable();  

    }   

    void getZhihuDailyNews(boolean forceUpdate, boolean clearCache, long date, @NonNull LoadZhihuDailyNewsCallback callback);   

    void getFavorites(@NonNull LoadZhihuDailyNewsCallback callback);    

    void getItem(int itemId, @NonNull GetNewsItemCallback callback);    

    void favoriteItem(int itemId, boolean favorite);    

    void saveAll(@NonNull List<ZhihuDailyNewsQuestion> list);   

}
```

[ZhihuDailyNewsDataSource.kt](https://github.com/TonnyL/PaperPlane/commit/45476a3ce62190e65cedceea4d4ae188f6b87e84#diff-10497b91725625d500f4ef39111ee113)

```kotlin
interface ZhihuDailyNewsDataSource {

    interface LoadZhihuDailyNewsCallback {

        fun onNewsLoaded(list: List<ZhihuDailyNewsQuestion>)

        fun onDataNotAvailable()

    }

    interface GetNewsItemCallback {

        fun onItemLoaded(item: ZhihuDailyNewsQuestion)

        fun onDataNotAvailable()

    }

    fun getZhihuDailyNews(forceUpdate: Boolean, clearCache: Boolean, date: Long, callback: LoadZhihuDailyNewsCallback)

    fun getFavorites(callback: LoadZhihuDailyNewsCallback)

    fun getItem(itemId: Int, callback: GetNewsItemCallback)

    fun favoriteItem(itemId: Int, favorite: Boolean)

    fun saveAll(list: List<ZhihuDailyNewsQuestion>)

}
```

Result 是一个 `sealed` (密封)类, 可能为 Success 或者 Error. 之前所有接受回调作为参数的方法, 现在都改成返回 Result 对象. 


[Result.kt](https://github.com/TonnyL/PaperPlane/blob/master/app/src/main/java/com/marktony/zhihudaily/data/source/Result.kt)

```kotlin
sealed class Result<out T : Any> {

    class Success<out T : Any>(val data: T) : Result<T>()

    class Error(val exception: Throwable) : Result<Nothing>()

}
```

然后我们来修改具体类中的实现方法, 首先是从本地数据库中获取数据, 如 `getZhihuDailyNews()`:

[ZhihuDailyNewsLocalDataSource.java#getZhihuDailyNews](https://github.com/TonnyL/PaperPlane/commit/45476a3ce62190e65cedceea4d4ae188f6b87e84#diff-3dd1ff342a4398326994cab4c6f28149L64)

```Java
@Override   
public void getZhihuDailyNews(boolean forceUpdate, boolean clearCache, long date, @NonNull LoadZhihuDailyNewsCallback callback) {   

    if (mDb == null) {  
        mDb = DatabaseCreator.getInstance().getDatabase();  
    }   

    if (mDb != null) {  
        new AsyncTask<Void, Void, List<ZhihuDailyNewsQuestion>>() { 

            @Override   
            protected List<ZhihuDailyNewsQuestion> doInBackground(Void... voids) {  
                return mDb.zhihuDailyNewsDao().queryAllByDate(date);    
            }   

            @Override   
            protected void onPostExecute(List<ZhihuDailyNewsQuestion> list) {   
                super.onPostExecute(list);  
                if (list == null) { 
                    callback.onDataNotAvailable();  
                } else {    
                    callback.onNewsLoaded(list);    
                }   
            }   

        }.execute();    
    }   
}
```

[ZhihuDailyNewsLocalDataSource.kt#getZhihuDailyNews](https://github.com/TonnyL/PaperPlane/blob/master/app/src/main/java/com/marktony/zhihudaily/data/source/local/ZhihuDailyNewsLocalDataSource.kt#L58)

```kotlin
override suspend fun getZhihuDailyNews(forceUpdate: Boolean, clearCache: Boolean, date: Long): Result<List<ZhihuDailyNewsQuestion>> = withContext(mAppExecutors.ioContext) {
    val news = mZhihuDailyNewsDao.queryAllByDate(date)
    if (news.isNotEmpty()) Result.Success(news) else Result.Error(LocalDataNotFoundException())
}
```

Repository 中之前的回调, 现在可以替换为 if 或 when 代码块:

[ZhihuDailyNewsRepository.java#getZhihuDailyNews](https://github.com/TonnyL/PaperPlane/commit/45476a3ce62190e65cedceea4d4ae188f6b87e84#diff-313b101331838cfed67d4f172c492427L73)

```Java
@Override   
public void getZhihuDailyNews(boolean forceUpdate, boolean clearCache, long date, @NonNull LoadZhihuDailyNewsCallback callback) {   

    if (mCachedItems != null && !forceUpdate) { 
        callback.onNewsLoaded(new ArrayList<>(mCachedItems.values()));  
        return; 
    }   

    // Get data by accessing network first. 
    mRemoteDataSource.getZhihuDailyNews(false, clearCache, date, new LoadZhihuDailyNewsCallback() { 
        @Override   
        public void onNewsLoaded(@NonNull List<ZhihuDailyNewsQuestion> list) {  
            refreshCache(clearCache, list); 
            callback.onNewsLoaded(new ArrayList<>(mCachedItems.values()));  
            // Save these item to database. 
            saveAll(list);  
        }   

        @Override   
        public void onDataNotAvailable() {  
            mLocalDataSource.getZhihuDailyNews(false, false, date, new LoadZhihuDailyNewsCallback() {   
                @Override   
                public void onNewsLoaded(@NonNull List<ZhihuDailyNewsQuestion> list) {  
                    refreshCache(clearCache, list); 
                    callback.onNewsLoaded(new ArrayList<>(mCachedItems.values()));  
                }   

                @Override   
                public void onDataNotAvailable() {  
                    callback.onDataNotAvailable();  
                }   
            }); 
        }   
    }); 

}
```

[ZhihuDailyNewsRepository.kt#getZhihuDailyNews](https://github.com/TonnyL/PaperPlane/blob/master/app/src/main/java/com/marktony/zhihudaily/data/source/repository/ZhihuDailyNewsRepository.kt#L58)

```kotlin
override suspend fun getZhihuDailyNews(forceUpdate: Boolean, clearCache: Boolean, date: Long): Result<List<ZhihuDailyNewsQuestion>> {
    if (!forceUpdate) {
        return Result.Success(mCachedItems.values.toList())
    }

    val result = mRemoteDataSource.getZhihuDailyNews(false, clearCache, date)
    return if (result is Result.Success) {
        refreshCache(clearCache, result.data)
        saveAll(result.data)

        result
    } else {
        mLocalDataSource.getZhihuDailyNews(false, false, date).also {
            if (it is Result.Success) {
                refreshCache(clearCache, it.data)
            }
        }
    }
}
```

最后, 我们在 Presenter 中调用时, 代码变成了线性的, 而不是之前的大段回调:

[ZhihuDailyPresenter.java#loadNews](https://github.com/TonnyL/PaperPlane/commit/45476a3ce62190e65cedceea4d4ae188f6b87e84#diff-08b6429078305c8a8faa33463b050754L55)

```Java
@Override   
public void loadNews(boolean forceUpdate, boolean clearCache, long date) {  

    mRepository.getZhihuDailyNews(forceUpdate, clearCache, date, new ZhihuDailyNewsDataSource.LoadZhihuDailyNewsCallback() {    
        @Override   
        public void onNewsLoaded(@NonNull List<ZhihuDailyNewsQuestion> list) {  
            if (mView.isActive()) { 
                mView.showResult(list); 
                mView.setLoadingIndicator(false);   
            }   
        }   

        @Override   
        public void onDataNotAvailable() {  
            if (mView.isActive()) { 
                mView.setLoadingIndicator(false);   
            }   
        }   
    }); 
}
```

[ZhihuDailyPresenter.kt#loadNews](https://github.com/TonnyL/PaperPlane/blob/master/app/src/main/java/com/marktony/zhihudaily/timeline/ZhihuDailyPresenter.kt#L46)

```kotlin
override fun loadNews(forceUpdate: Boolean, clearCache: Boolean, date: Long) = launchSilent(uiContext) {
    val result = mRepository.getZhihuDailyNews(forceUpdate, clearCache, date)
    if (mView.isActive) {
        if (result is Result.Success) {
            mView.showResult(result.data.toMutableList())
        }

        mView.setLoadingIndicator(false)
    }
}
```

到这里, 使用 Coroutines 替代 Callback 的工作基本完成. 迁移到 Kotlin Coroutines 使得代码更加干净也更加容易理解了. 

## 学习 Kotlin 的资源

+ [Kotlin 官方网站]( https://kotlinlang.org/ ) 以及 [Kotlin 语言中文站](https://www.kotlincn.net/)

+ [JetBrains TV](https://www.youtube.com/channel/UCGp4UBwpTNegd_4nCpuBcow) 以及 [Android Developers](https://www.youtube.com/channel/UCVHFbqXqoYvEWM1Ddxl0QDg) 的 Youtube Channel: JetBrains TV 会分享 KotlinConf 大会的记录视频, 而 Android Developers 会不定时分享一些 Kotlin 使用小技巧
+ [Kotlin Slack](http://slack.kotlinlang.org/): JetBrains 官方的人和很多开发者聚集在这里, 遇到问题时可以向他们请教

## 参考文章
<a name="reference-1">[1]</a> 来源: [https://www.kotlincn.net/docs/reference/comparison-to-java.html#kotlin-%E6%9C%89%E8%80%8C-java-%E6%B2%A1%E6%9C%89%E7%9A%84%E4%B8%9C%E8%A5%BF]()

<a name="reference-2">[2]</a> 关于项目的开发历程, 请参考[如何用一周时间开发一款 Android APP 并在 Google Play 上线](https://tonnyl.github.io/Develop-an-Android-App-and-Publish-it-on-Google-Play-in-a-Week/), 关于应用中全局字体的应用和夜间模式的实现, 请分别参考: [简单高效的实现 Android App 全局字体替换](https://tonnyl.github.io/Apply-Global-Fonts-With-Fonts-in-XML/), [简洁优雅地实现夜间模式](https://tonnyl.github.io/Night-Mode-on-Android/).

<a name="reference-3">[3]</a> 协程的官方文档: http://kotlinlang.org/docs/reference/coroutines.html

本文参考了:

[Migrating todo-mvp-kotlin to coroutines](https://proandroiddev.com/migrating-todo-mvp-kotlin-to-coroutines-d71f6f815e44), 作者 [Dmytro Danylyk](https://proandroiddev.com/@dmytrodanylyk?source=post_header_lockup), 是一名 Android Google Developer Expert (GDE).

本文由 TonnyL 创作, 发表在 [Tonny’s Blog](https://tonnyl.github.io/) , 转载请遵守 **CC BY-NC-ND 4.0** 协议.