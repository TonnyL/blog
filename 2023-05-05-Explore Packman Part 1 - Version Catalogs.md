# Explore Packman Part 1 - Version Catalogs

> 本篇为 Packman 项目开发总结系列的第一篇，这个系列主要介绍在开发 Packman 项目过程中所使用的新技术、遇到的问题及其解决办法。
> 

> Packman 的源代码地址为 [https://github.com/TonnyL/Packman](https://github.com/TonnyL/Packman)，每一篇文章所涉及的内容与源码对应参考更加方便。
> 

## 依赖版本管理的历史发展

1. [硬编码](https://www.notion.so/Explore-Packman-Part-1-Version-Catalogs-9cf56d5689574549950edeac4089f98f)
2. [gradle ext 或 gradle.properties](https://www.notion.so/Explore-Packman-Part-1-Version-Catalogs-9cf56d5689574549950edeac4089f98f)
3. [buildSrc 或 Composing Builds](https://www.notion.so/Explore-Packman-Part-1-Version-Catalogs-9cf56d5689574549950edeac4089f98f)
4. [version catalogs](https://www.notion.so/Explore-Packman-Part-1-Version-Catalogs-9cf56d5689574549950edeac4089f98f)

### 硬编码

```groovy
// build.gradle (module)
dependencies {
    implementation "org.jetbrains.kotlin:kotlin-stdlib:1.8.20"
    implementation 'androidx.appcompat:appcompat:1.7.0-alpha01'
}
```

在项目只有一个模块（module）的情况下，直接将依赖版本硬编码是可行的。硬编码方式：

👍 使用简单；

👍 Android Studio 在依赖有更新版本时**有**提示；

👎 不支持代码提示、补全、跳转（Groovy 脚本的问题）；

👎 没有统一依赖版本，不利于维护（在多模块项目下尤其明显）。

### gradle ext 或 gradle.properties

```groovy
// build.gradle (project)
ext {
    versions = [
        kotlin_version: "1.8.20",
        appcompat     : "1.7.0-alpha01"
    ]

    libs = [
        kotlin_stdlib: "org.jetbrains.kotlin:kotlin-stdlib:${versions.kotlin_version}",
        appcompat:"androidx.appcompat:appcompat:${versions.appcompat}"
    ]
}

// build.gradle (module)
dependencies {
    implementation libs.kotlin_stdlib
    implementation libs.appcompat
}
```

将依赖版本定义在 gradle ext、独立的 .gradle 文件或 gradle.properties 文件中，思路其实是一样的，均是统一版本信息，并将其集中管理。这种方式：

👍 统一了依赖版本；

👍 Android Studio 在依赖有更新版本时**有**提示；

👎 不支持代码提示、补全、跳转（Groovy 脚本的问题）。

### buildSrc 或 Composing Builds

```kotlin
// buildSrc/src/main/java/Dependencies.kt
object Versions {
    const val kotlin = "1.8.20"
    const val appcompat = "1.7.0-alpha01"
}

object Libs {
    const val kotlin_stdlib = "org.jetbrains.kotlin:kotlin-stdlib:${Versions.kotlin}" core_ktx = "androidx.core:core-ktx:1.3.2:${Versions.core_ktx}"
    const val appcompat = "androidx.appcompat:appcompat:${Versions.appcompat}"
}

// build.gradle.kts (module)
dependencies {
    implementation(Libs.kotlin_stdlib)
    implementation(Libs.appcompat)
}
```

在 Gradle 5.0 之后，Gradle 支持了使用 Kotlin 编写构建逻辑，并且现在已经是[默认语言选项](https://blog.jetbrains.com/kotlin/2023/04/kotlin-dsl-is-the-default-for-new-gradle-builds/)了。关于如何从 Groovy 迁移到 Kotlin DSL，可以参考文章 [从 Groovy 到 Kotlin DSL, Android 构建脚本迁移指南](https://lizhaotailang.works/migrate-groovy-script-to-kotlin-dsl/) 。

Composing Builds 可以简单理解为 buildSrc 的增强版本（后面的文章会详细介绍他们的相同与不同，敬请关注）。这种方式：

👍 统一了依赖版本；

👍 支持代码提示、补全、跳转；

👎 Android Studio 在依赖有更新版本时**没有**提示（值得一提的是，类似于 [https://github.com/Splitties/refreshVersions](https://github.com/Splitties/refreshVersions) Gradle 插件提供了一种曲线救国的方式，实现了版本更新的检测）。

## 什么是 Version Catalogs

版本目录，是 Gradle 7.0 引入的一种管理依赖版本的方式。在有多个模块的项目中，统一的依赖版本管理是非常有必要的。这种方式：

👍 统一了依赖版本；

👍 Android Studio 在依赖有更新版本时**有**提示；

👍 支持代码提示、补全、跳转。

## 如何使用 Version Catalogs

### 使用 libs.versions.toml

默认情况下从创建 version catalogs 文件开始。在项目的 `gradle` 目录下创建一个名为 `libs.versions.toml` 的文件，Gradle 默认会搜索读取此文件并自动配置（当然你也可以自定义此文件的名称，但这就需要相应修改构建脚本逻辑了）。

`libs.versions.toml` 主要包含4个部分：

```toml
# 声明版本号，能在 [libraries] 和 [plugins] 部分被引用
[versions]
androidxAppCompat = "1.7.0-alpha01"
androidxLifecycle = "2.6.0-alpha04"
compose = "1.4.0"
compileSdk = "33"

# 声明依赖
[libraries]
androidx-appcompat = { group = "androidx.appcompat", name = "appcompat", version.ref = "androidxAppCompat" }
androidx-lifecycle-runtime-compose = { group = "androidx.lifecycle", name = "lifecycle-runtime-compose", version.ref = "androidxLifecycle" }
androidx-lifecycle-viewmodel-ktx = { group = "androidx.lifecycle", name = "lifecycle-viewmodel-ktx", version.ref = "androidxLifecycle" }
androidx-lifecycle-viewmodel-compose = { group = "androidx.lifecycle", name = "lifecycle-viewmodel-compose", version.ref = "androidxLifecycle" }

# 声明依赖组，一组依赖如果具有相同版本号则可以声明为 bundles
[bundles]
androidx-lifecycle = ["androidx-lifecycle-runtime-compose", "androidx-lifecycle-viewmodel-ktx", "androidx-lifecycle-viewmodel-compose"]

# 声明插件
[plugins]
compose-multiplatform = { id = "org.jetbrains.compose", version.ref = "compose" }
```

除了在 `libs.versions.toml` 文件中声明依赖外，version catalogs 还可以在 `settings.gradle.kts` 文件中声明。上面的声明等价于：

```kotlin
// settings.gradle.kts
dependencyResolutionManagement {
		versionCatalogs {
        getByName("libs") {
            version("androidxAppCompat", "1.7.0-alpha01")
            version("androidxLifecycle", "2.6.0-alpha04")
            version("compose", "1.4.0")
            version("compileSdk", "33")

            library(
                "androidx-appcompat",
                "androidx.appcompat",
                "appcompat"
            ).versionRef("androidxAppCompat")
            library(
                "androidx-lifecycle-runtime-compose",
                "androidx.lifecycle",
                "lifecycle-runtime-compose"
            ).versionRef("androidxLifecycle")
            library(
                "androidx-lifecycle-viewmodel-ktx",
                "androidx.lifecycle",
                "lifecycle-viewmodel-ktx"
            ).versionRef("androidxLifecycle")
            library(
                "androidx-lifecycle-viewmodel-compose",
                "androidx.lifecycle",
                "lifecycle-viewmodel-compose"
            ).versionRef("androidxLifecycle")

            bundle(
                "androidx-lifecycle",
                listOf(
                    "androidx-lifecycle-runtime-compose",
                    "androidx-lifecycle-viewmodel-ktx",
                    "androidx-lifecycle-viewmodel-compose"
                )
            )

            plugin("compose-multiplatform", "org.jetbrains.compose").versionRef("compose")
        }
    }
}
```

由此也可瞥见声明 version catalogs 的一些规则：

- 依赖需要关联一个别名，依赖本身以`group`、`artifact`和`version` 的格式表示；
- 别名只能由一组以破折号（ `-` ，推荐）、下划线（`_`）和点（`.`）分隔的标识符组成。标识符本身只能是 ascii 字符，小写最好，结尾为数字；
- 用于分隔标识符的几个标点符号最终会被映射为 `.`：比如 `compose-multiplatform` 会自动转换为 `compose.multiplatform` 。如果想要避免转换可以使用大小写区分，例如 `androidx-lifecycleRuntimeCompose` 会转换为 `androidx.lifecycleRuntimeCompose` ；
- 依赖可以声明为 `group + artifact` 的形式（`group = "androidx.appcompat", name = "appcompat"`），也可以直接声明为一个独立字符串 module（`module = "androidx.appcompat:appcompat"` ）；
- 依赖版本可以定义成一个字符串，也可以定义为一个范围：
    
    ```toml
    [versions]
    androidxAppCompat = { strictly = "[1.6.1, 1.7.0-alpha01[", prefer = "1.7.0-alpha01" }
    ```
    

version catalogs 配置好后，在构建脚本中就可以使用对应的依赖了：

```kotlin
// build.gradle.kts(module)
plugins {
	id("com.android.application")
	// 引用声明的插件
	alias(libs.plugins.compose.multiplatform)
}

android {
	compileSdk = libs.versions.compileSdk.get().toInt()
}

dependencies {
    implementation(libs.androidx.appcompat)

    implementation(libs.bundles.androidx.lifecycle)
}
```

version catalogs 提供了一个类型不安全的 API，用于在构建脚本中对其访问。

```kotlin
// build.gradle.kts(module)
val versionCatalogs = extensions.getByType<VersionCatalogsExtension>().named("libs")
dependencies {
    versionCatalogs.findLibrary("androidx-appcompat").ifPresent {
        implementation(it)
    }
}
```

### 自定义 toml 文件

除了使用默认的 `libs.versions.toml` 文件外，我们还可以修改其文件名和路径：

```kotlin
// settings.gradle.kts
dependencyResolutionManagement {
    versionCatalogs {
        create("libraries") {
            from(files("../gradle/libraries.versions.toml"))
        }
    }
}
```

这种方式也适用于 version catalogs 搭配 buildSrc 使用的情况，从而实现主项目和 buildSrc 依赖声明的复用。

## 跨项目共享 version catalogs

尽管从本地文件中导入 version catalogs 很方便，但它没有解决团队内跨项目或者与外部用户统一版本目录的问题。在没有 version catalogs 之前，我们可以通过将各项依赖配置在一个独立项目中，并将其作为 Gradle 插件发布在 Gradle plugin portal 或者内部仓库中，让其他用户应用这个插件即可。

有了 version catalogs 后，Gradle 提供了一个同名插件，配合 `maven-publish` 插件可以很方便的实现发布 version catalogs。

```kotlin
// build.gradle.kts
// 1. 应用 version catalog 插件
plugins {
    `version-catalog`
    `maven-publish`
}

// 2. 定义 catalog
catalog {
    // 在此声明依赖、依赖组、版本号等
    versionCatalog {
        library("my-lib", "com.mycompany:mylib:1.2")
    }
}

// 3. 发布 catalog
publishing {
    publications {
        create<MavenPublication>("maven") {
            from(components["versionCatalog"])
        }
    }
}

// settings.gradle.kts
// 发布完成后就可以导入已发布的 catalogs 
dependencyResolutionManagement {
    versionCatalogs {
        create("libs") {
            from("com.mycompany:catalog:1.0")

						// 仍然可以复写 com.mycompany:catalog:1.0 中定义的版本
						version("groovy", "3.0.6")
        }
    }
}
```

## Version Catalogs VS Platform

BOM（[bill of materials](https://maven.apache.org/guides/introduction/introduction-to-dependency-mechanism.html#Importing_Dependencies)） 是 Maven 中的概念，简单来说，BOM **定义**（注意只是定义，并没有引入）了一系列的依赖及其版本。在 Android 开发中，Compose 支持通过 BOM 添加依赖。

```kotlin
dependencies {
    // 引入 Compose BOM
		implementation(platform("androidx.compose:compose-bom:2023.04.01"))

    // 使用预发布版本覆盖 Compose BOM 中定义的 Material Design 3 库
    implementation("androidx.compose.material3:material3:1.1.0-alpha01")

    // 引入其他 Compose 库而不需要声明具体版本号
    implementation("androidx.compose.foundation:foundation")
}
```

version catalogs 与 platform 都和依赖版本相关，也都用于在项目中共享依赖版本，那两者的使用场景是什么呢？简单来说：

- catalogs 只用于定义项目中的依赖及其版本，并生成类型安全的访问器；
- platform 为依赖图（dependency graph）应用特定版本，并以此影响依赖解析。

简单来说，catalogs 正如它名字所示的那样，更关心依赖是如何被声明的，以及声明了哪一些依赖，它们的版本是什么（Gradle 官方视 catalogs 为一项非常好的工程实践，推荐你在所有场景下都使用它声明依赖）；而 platform 更关心依赖的版本信息，你声明了哪些依赖它并不关心。实际上，他们并不是对立的，你可以同时使用 version catalogs 和 platform ：

```toml
# libs.versions.toml
[versions]
composeBom = "2022.10.00"

[libraries]
androidx-compose-bom = { group = "androidx.compose", name = "compose-bom", version.ref = "composeBom" }
```

```kotlin
// build.gradle.kts
dependencies {
		implementation(platform(libs.androidx.compose.bom))
    implementation("androidx.compose.ui:ui")
}
```

## Version Catalogs 存在的不足

- 目前和 buildSrc 搭配使用仍存在一定的问题：默认配置下 version catalogs 目前是没有办法直接在 buildSrc 中访问的（参考 https://github.com/gradle/gradle/issues/15383），当然，buildSrc 本身存在着这样那样的问题（在本系列的其他文章会详细说明），但是并不妨碍我喜欢使用它。
- 不支持在依赖版本后追加 `@aar` 标记的方式（参考 https://github.com/gradle/gradle/issues/20074），解决办法为：
    
    ```kotlin
    dependencies {
    		api(libs.abc) {
            artifact {
                type = "aar"
            }
        }
    }
    ```
    
- `[versions]` 部分不支持声明数字，只支持字符串（上文有提到），当然可以通过迂回的方式解决：
    
    ```toml
    [versions]
    compileSdk = "33"
    ```
    
    ```kotlin
    android {
    	compileSdk = libs.versions.compileSdk.get().toInt()
    }
    ```
    

事实上，相对于老前辈们，version catalogs 已经足够优秀了，以上的问题更多的是无伤大雅的小问题，并不严重影响使用。

## 参考链接

- Sharing dependency versions between projects - [https://docs.gradle.org/current/userguide/platforms.html](https://docs.gradle.org/current/userguide/platforms.html)