# Kotlin 扩展方法进化之 Context Receiver

## 为什么会有 Context Receiver

我们都爱扩展方法。在介绍 Context Receiver 之前，我们先了解一下 Kotlin 目前的扩展方法实现存在的问题。举个例子，在 Android 中，我们通常都会写一个扩展方法将 int 转换为 dp:

```kotlin
fun Int.dp(context: Context): Int {
    // ...
}
```

还有另外一种方式:

```kotlin
fun Context.dp(value: Int): Int {
    // ...
}
```

那么，首先，为什么会存在上面两种实现方式呢？因为目前 Kotlin 的扩展方法或者属性只支持扩展在一个类或者接口上；其次，上面这两种扩展方法，哪种更符合直觉呢？我认为两种扩展方式都符合，或者说，都不符合，为什么呢，我们可以这样理解上面的扩展方法：只有在同时有 `Context` 和 `Int` 两个参与者的情况下，`dp()` 这个扩展方法才能够被使用。

我们假设一种实现方式：

```kotlin
fun (Context, Int).dp(): Int {
	// ...
}
```

目前的 Kotlin 实现是不支持这样的定义的（实际上这也是 Kotlin 团队[最初设想的实现方式](https://youtu.be/0FF19HJDqMo?t=871)）。不过这种方式是否比上面的两种实现更加符合直觉呢？

再举一个例子：

```kotlin
interface Entity

interface Scope {
    fun Entity.doAction() { // 成员扩展方法
        // ...
    }
}

class ScopeImpl: Scope {

    init {
        val entity = object : Entity { }
        entity.doAction()
    }

}

class EntityImpl: Entity {

    init {
        val entity = object : Entity { }
        this.doAction() //  <- 错误
        entity.doAction() // <- 错误
    }

}

```

上面的例子展示了成员扩展方法（[member extension function](https://kotlinlang.org/docs/extensions.html#declaring-extensions-as-members)）是如何工作的。那我们为什么需要成员扩展方法呢？简单来说就是限制扩展方法能够被谁调用。示例中的 `doAction` 方法只有在 `Scope` 类及其子类中才能被调用。事实上，成员扩展属性和方法可以声明为 `open` 并在子类中被复写的（回想一下扩展方法的实现原理）。

我们用成员扩展方法改造一下文章开头的 `dp` 方法：

```kotlin
class Context {

    fun Int.dp(): Int {
        // ...
    }

}
```

因为 `Context` 类在 Android 中是在 SDK 里定义的，我们并不能随意修改，所以上面的实现只是一个美好愿望，实际上并不能实现。

OK，到这里，我们可以总结一下 Kotlin 的扩展方法和属性目前存在的一些限制，或者说，Context Receiver 要解决什么问题：

1. 首先，只支持扩展在单一的类或者接口上，这就限制了多种抽象的组合能力，我们也不能声明一个方法，它只有在多个条件都满足的情况下，才能被调用。而这导致了实现一个功能有好几种方式（虽然在 Kotlin 中这种现象非常常见，但这显然和 Python 之禅中 “应该提供一种，且最好只提供一种，一目了然的途径”的理念不太一致），并且这几种实现方式都不完美，造成重复实现以及增加理解难度；
1. 其次，成员扩展方法想法很好，但是现实太骨感，它的应用范围太窄，在遇到第三方类的时候就无能为力了，这也就限制了它在大型应用中解耦合、模块化和结构化 API 的能力；
1. 最后，成员扩展方法始终是一个扩展，它只能通过 `entity.doAction()` 的方式调用，而一些方法调用不应该依赖特定的实例，也不应该这样声明。而现在又没有一种方式，可以在特定的上下文作用域内，直接声明 **顶级方法（top-level function）**，通过 `doAction()` 的方式调用。

## 什么是 Context Receiver

Kotlin 在 Kotlin [1.6.20](https://blog.jetbrains.com/kotlin/2022/02/kotlin-1-6-20-m1-released/#prototype-of-context-receivers-for-kotlin-jvm) 版本引入了在 Kotlin/JVM 上的 Context Receiver 的原型实现，解决了上面提到的问题。它目前仍然处于[草案阶段](https://github.com/Kotlin/KEEP/issues/259)。可以在其[设计草案](https://github.com/Kotlin/KEEP/blob/master/proposals/context-receivers.md)中查看语法和详细介绍。

官方示例是这样的：

```kotlin
interface LoggingContext {
    val log: Logger // LoggingContext 作为上下文提供了一个 Logger 的引用
}

context(LoggingContext)
fun startBusinessOperation() {
    // 你可以访问 log 字段，因为 LoggingContext 是一个隐式的接收者
    log.info("Operation has started")
}

fun test(loggingContext: LoggingContext) {
    with(loggingContext) {
        // 在作用域内，你需要有 LoggingContext 作为隐式接收者来调用 startBusinessOperation()
        startBusinessOperation()
    }
}
```

## 如何使用 Context Receiver

因为目前 Context Receiver 处于实验阶段，所以默认是没有开启的。要使用这项功能，我们需要在模块的 **build.gradle** 或者 **build.gradle.kts** 构建脚本中增加额外的编译器参数 **-Xcontext-receivers** ：

```kotlin
tasks.withType<KotlinCompile>().configureEach {
    kotlinOptions {
        jvmTarget = JavaVersion.VERSION_1_8.toString()
        freeCompilerArgs = freeCompilerArgs + listOf(
            "-Xcontext-receivers"
        )
    }
}
```

注意事项：

+ 在启用 `-Xcontext-receivers` 的情况下，编译器将生成不能用于生产代码的预发布二进制文件；
+ 目前，IDE 对上下文接收器的支持并不完善。

讲完注意事项，我们仿照官方示例，用 Context Receiver 实际改造一下之前的 `dp` 方法。

```kotlin
context(Context, Int)
fun dp(): Int {
	// ...
}
```

小菜一碟，再看看如何调用：

```kotlin
class MainActivity : ComponentActivity() {

	override fun onCreate(savedInstanceState: Bundle?) {
		// scope 1
		val oneDp = with(1) {
			// scope 2
			dp()
		}
	}

}
```

为什么要显式的使用 `with(1)` 这个看起来很奇怪的写法？回想一下 Java 中 `this` 关键字的作用。`scope1` 处 `this` 指向的是 `MainActivity` 的对象，是 `Context` 的作用域（注意不是 Android 中的 `android.content.Context`）；`scope2` 处 `this`  指向的是 `1` 这个对象，是 `Int` 的作用域。我们也可以在 Context Receiver 中切换上下文对象。

```kotlin
context(Context, Int)
fun dp(): Int {
	println(this@Context)
	println(this@Int)
	// ...
}
```

## Context Receiver 与 Java 代码的交互
我们先回想一下 Context Receiver 之前的扩展方法是如何实现和 Java 交互的：

```kotlin
// Extensions.kt
fun Context.dp(value: Int): Int {
	// ...
}

context(Context, Int)
fun dp(): Int {
	// ...
}
```

反编译后的 Java 代码：

```java
public final class ExtensionsKt {

	// Context Receiver
	public static final int dp(Context $this$dp, int $this$dp1) {
		// ...
   	}

	// 普通扩展方法
	public static final int dp(@NotNull Context $this$dp, int value) {
		Intrinsics.checkNotNullParameter($this$dp, "$this$dp");
		// ...
	}
}
```

为什么反编译后的代码没有太大区别？因为 Context Receiver 本质上还是扩展方法，只是扩展方法的 pro plus 版本而已。那下面的代码反编译后的 Java 代码会有区别吗？

```kotlin
interface Ext {

    context(Context)
    fun abc(): Int

    fun Context.abc2(): Int

}
```

## Context Receiver 之“箭头型”代码

```kotlin
context(Foo, Bar, Baz, Qux, Quux, Corge, Grault, Garply, Waldo, Fred, Plugh, Xyzzy, Thud)
fun function() {
}

with(foo) {
    with(bar) {
        with(baz) {
            with(qux) {
                with(quux) {
                    with(corge) {
                        with(grault) {
                            with(garply) {
                                with(waldo) {
                                    with(fred) {
                                        with(plugh) {
                                            with(xyzzy) {
                                                with(thud) {
                                                    function()
                                                }
                                            }
                                        }
                                    }
                                }
                            }
                        }
                    }
                }
            }
        }
    }
}
```

这段代码有夸张的嫌疑，而且我想到了一个计算机领域的经典笑话：
> 据说某俄国特工九死一生偷到了NASA太空火箭发射程序源代码的最后一页，代码 是：
> > ))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))))
> 有人在讲这个笑话的时候说是}}}}}}}}，觉得是黑C/C++，但其实这个段子是用来调侃lisp的，不然也没必要套在NASA头上。不过现在NASA都用Python了（所以最后一页会偷到大片的空格吗……）
>
> https://www.zhihu.com/question/20034686/answer/449134099

如果 NASA 也用 Kotlin，还赶新潮地用上了 Context Receiver ，那俄国特工偷到的就是一整页的花括号了（如何同时黑俄国特工、NASA 和 Kotlin 🤣）。

当然，这样的代码并不算“箭头形”代码，只是形似，并不是神似。事实上，Kotlin 团队是非常不支持你在 `context()` 中写一长串参数的：

```kotlin
context(Foo, Bar, Baz, Qux, Quux, Corge, Grault, Garply, Waldo, Fred, Plugh, Xyzzy, Thud) // 坏的实践，太多独立的上下文
fun function() {
}
```

他们推荐的[方案](https://github.com/Kotlin/KEEP/blob/master/proposals/context-receivers.md#designing-context-types)是：

```kotlin
interface TopLevelContext: Foo, Bar, Baz, Qux, Quux, Corge, Grault, Garply, Waldo, Fred, Plugh, Xyzzy, Thud

context(TopLevelContext) // 好的实践
fun function() {
}
```

那么问题来了，如果我的确需要用到很多独立上下文，比如前面提到的 `dp` 方法就需要 `Context` 和 `Int`，应该如何抽象这个 `TopLevelContext` 呢？

1. 抽象为接口

	```kotlin
	interface TopLevelContext: Context, Int
	```

	 并不能成功，Context 为 abstract class, Int 为 final class

2. 抽象为抽象类

	```kotlin
	abstract class TopLevelContext: Context(), Int()
	```

	也不能成功，不支持同时继承多个类，且 Int 为 final class

所以，按照 Kotlin 团队推荐的方案并不能成功抽象出 `TopLevelContext`。如果要你来解决这个问题，你会怎么做呢？一个可能的思路是 `with` 支持多参数：

```kotlin
with(foo, bar, baz) {
  // ...
}
```

## 结论
在尝试过 Context Receiver 后，我相信它在不远的将来就会稳定下来，然后在帮助工程师改善代码质量的同时被滥用（参考扩展方法的使用现状，扩展方法是一个好的特性，但也请不要为了扩展而扩展，参考[Bad Kotlin Extensions](https://krossovochkin.com/posts/2021_01_25_bad_kotlin_extensions/)）。

另外，非常建议读一下 Context Receiver 的 [KEEP 文档](https://github.com/Kotlin/KEEP/blob/master/proposals/context-receivers.md)，里面详细记录了这样一个功能是如何设计和实现的，读完之后相信你会对 Kotlin 这门语言有更加深入的理解。