# Benchmarking Android Build With Gradle Profiler

## 为什么需要基准测试

Gradle 的构建是一个非常复杂的过程，影响构建时间的因素也非常多，比如硬件配置、缓存、守护进程的状态、网络状况等等（可以参考 [Android Gradle 构建速度为什么这么慢，还有救吗？](https://lizhaotailang.works/why-android-gradle-slow/)）。而随着项目越来越大，或自定义构建逻辑越来越多，我们可能需要

1. 持续监控构建的恶化情况；
2. 更加深入地了解构建过程中遇到的瓶颈；
3. 验证优化构建参数后的实际提升效果等；

这时候就需要对 Gradle 构建进行基准测试。

## 什么是 Gradle Profiler

想象一下，我们想要知道 Gradle 构建在某个特定场景下能有多快，要怎么做呢：

![Untitled](https://pica.zhimg.com/80/v2-0fea7ba1e4a34fd0d00aa5f5f9398789_1440w.png)

正如上面提到的，Gradle 构建涉及的环境因素非常多，单次运行并不一定能准确反映构建有多快，于是我们需要循环上面的过程，最后计算平均时间。整个过程非常无聊且耗时，最难受的是需要人监控记录，并手动触发下一个循环。

这个过程就是 [Gradle Profiler](https://github.com/gradle/gradle-profiler) 的用武之地。Gradle Profiler 作为一个自动化工具，用于收集 Gradle 构建的性能分析和基准测试信息。使用 Gradle Profiler，上面复杂的任务只需要开发者启动一个命令、无需其他操作的情况下即可完成。

## 安装 Gradle  Profiler

- Homebrew: `brew install gradle-profiler`
- SDKMAN!: `sdk install gradleprofiler`
- 二进制：[https://github.com/gradle/gradle-profiler/releases](https://github.com/gradle/gradle-profiler/releases)

## 运行基准测试

```bash
gradle-profiler --benchmark --project-dir <root-dir-of-build> <task>...
```

- `<root-dir-of-build>` 是需要对构建进行基准测试的项目的目录；
- `<task>` 是需要运行的 Gradle 任务的名称，和运行 `gradle` 命令时所使用的相同。

## 分析构建结果

构建任务运行完成后，测试结果会写入 `profile-out/` 目录下的 `profile-out/benchmark.csv` 和 `profile-out/benchmark.html` 两个文件中，分别提供基础的 .csv 格式的数据和基于此数据的网页报告。

`benchmark.csv`文件的内容大概为：

| version | Gradle 7.4 |
| --- | --- |
| tasks | assembleDebug |
| value | total execution time |
| warm-up build #1 | 307383 |
| warm-up build #2 | 38684 |
| warm-up build #3 | 35586 |
| warm-up build #4 | 34103 |
| warm-up build #5 | 35617 |
| warm-up build #6 | 35248 |
| measured build #1 | 35303 |
| measured build #2 | 34144 |
| measured build #3 | 35492 |
| measured build #4 | 36017 |
| measured build #5 | 34656 |
| measured build #6 | 35229 |
| measured build #7 | 38136 |
| measured build #8 | 38152 |
| measured build #9 | 38974 |
| measured build #10 | 34507 |

可以看到第一次预热（warm-up）构建花费的时间比其他构建都要长很多，这是因为在默认情况下，gradle-profiler 在测量构建时间时会使用预热之后的 [Gradle 守护进程（Gradle daemon ）](https://docs.gradle.org/current/userguide/gradle_daemon.html)。Gradle 守护进程运行在后台，用于减少重复启动 JVM 所带来的消耗，提升 Gradle 构建的性能。所以第一次预热构建所花费时间会更长，因为要启动 Gradle daemon 。抛开预热构建不谈，只看标准构建的话还是很符合预期的，没有很大波动。

`benchmark.html` 在 `benchmark.csv` 的基础上提供了更加直观、信息更丰富且可交互的网页结果。

![Untitled](https://picx.zhimg.com/80/v2-d78890f410c4db4cc4ee4e68f72af692_1440w.png)

除了以图表形式呈现出的构建时间外，我们还可以看到：

- 标准构建所花费时间的平均数、中位数、标准差和其他统计数据；
- Gradle 参数信息；
- JVM 参数等等。

## 自定义测试场景

上面[开启基准测试](https://www.notion.so/Benchmarking-Android-Build-With-Gradle-Profiler-faed1591e407477e8dbced223ceb7c8e)所使用的命令行只指定了项目目录和所要运行的 Gradle 任务名称，其他参数均为默认，实际上还有很多可定制项我们可以配置：

- `--output-dir <dir>`: 写入测试结果的目录，默认值为 `profile-out` 。如果 `profile-out` 目录已经存在了，就会尝试 `profile-out-<index>` 增加索引的方式新建目录。
- `--warmups`: 指定预热构建的运行次数，基准测试下默认为6，不启用预热守护进程的情况下则为1。
- `--iterations`: 执行标准构建的运行次数，基准测试下默认为10。
- `--bazel`、`--buck`、`--maven`: 使用 bazel、buck 或者 maven 而不是 Gradle 进行构建。

下面的命令行参数只有在使用 Gradle 构建时生效：

- `--gradle-user-home`: Gradle 用户根目录，为了在性能测试时和其他构建隔离，默认为 `<project-dir>/gradle-user-home` 。
- `--gradle-version <version>`: 指定用于运行构建的 Gradle 版本，覆盖默认的版本。一次可以指定多个版本。
- `--no-daemon`: 运行构建时不启用 Gradle 守护进程，默认是会启用守护进程。
- `--cold-daemon`: 使用不经预热、每次都冷启动的守护进程。 默认是会使用预热的守护进程。

这里只列出了部分配置选项，完整列表可以参考 [https://github.com/gradle/gradle-profiler#command-line-options](https://github.com/gradle/gradle-profiler#command-line-options) 。

如果全部选项均通过命令指定，那这个命令就会变得非常复杂了，而且不易管理。我们可以通过定义一个 `.scenarios` 文件解决，`.scenarios` 可以定义更加复杂的场景，然后应用这个文件即可：

```bash
gradle-profiler --benchmark --project-dir . --scenario-file performance.scenarios
```

`.scenarios` 文件定义遵守 [https://github.com/lightbend/config](https://github.com/lightbend/config) 规则，下面为一个示例：

```
# performance.scenarios
clean_build {
    tasks = ["assembleDebug"]
    cleanup-tasks = ["clean"]
    gradle-args = ["--no-build-cache"]
    warm-ups = 3
}

inc_build {
    tasks = ["assembleDebug"]
    apply-android-layout-change-to = "app/src/main/res/layout/activity_main.xml"
    apply-abi-change-to = "app/src/main/java/com/example/app/MainActivity.kt"
    apply-non-abi-change-to = "app/src/main/java/com/example/app/MainFragment.kt"
    clear-build-cache-before = SCENARIO
    warm-ups = 3
}
```

如果一个 `.scenarios` 文件中定义了多个场景，我们也可以只运行其中某个场景：

```
gradle-profiler --benchmark --project-dir . --scenario-file performance.scenarios clean_build
```

## 增量构建的基准测试

在现实世界的日常开发过程中，我们实际运行增量构建的次数应该是远多于全量构建的，而 Gradle Profiler 也支持对增量构建进行基准测试。

- 更改构建逻辑：
    - `apply-build-script-change-to`：在 Groovy 或 Kotlin DSL 脚本中（build.gradle(.kts)、init.gradle(.kts)、settings.gradle(.kts) 等）增加逻辑，每次运行测试时都会将前一次测试增加的逻辑删除并添加新的逻辑。
    - `apply-project-dependency-change-to` ：在 Groovy 或 Kotlin 构建脚本中添加项目依赖，每次运行测试时都会将前一次测试增加的依赖删除并添加新的依赖。
    - `apply-property-resource-change-to` ：在 properties 文件中增加一个配置项，每次运行测试都会将前一次测试增加的配置项删除并新增一个配置项。
- 更改 Kotlin 或 Java 代码：
    - `apply-abi-change-to` ：为 Kotlin 或 Java 类新增一个 public 方法，每次运行测试都会将前一次测试增加的方法删除并新增一个 public 方法。
    - `apply-non-abi-change-to` ：更改 Kotlin 或 Java 类中一个 public 方法的方法体（function body），但是不改变这个方法的签名。
- 更改 C/C++ 代码：
    - `apply-h-change-to` ：在 C/C++ 头文件中增加一个方法，每次运行测试都会将前一次添加的方法声明删除并添加一个新的方法。
    - `apply-cpp-change-to` ：在 C/C++ 源文件中新增一个方法，每次运行测试都会将前一次添加的方法声明删除并添加一个新的方法。
- 更改和 Android 相关的文件：
    - `apply-android-resource-change-to` ：在 Android 资源文件中新增一条字符串，每次运行测试都会将前一次新增的字符串删除并添加一条字符串。
    - `apply-android-resource-value-change-to` ：更改 Android 资源文件中的字符串。
    - `apply-android-manifest-change-to` ：在 Android 清单文件中新增一条权限声明。
    - `apply-android-layout-change-to` ：在 Android 布局文件中添加一个不可见的 view ，即支持传统的布局，也支持 Databinding 布局。
    - `apply-kotlin-composable-change-to` ：在 Kotlin 源文件中新增一个 `@Composable` 方法。
- 其他情况：
    - `clear-build-cache-before` ：在场景执行前（`SCENARIO`） 、在清理工作执行前（`CLEANUP`）亦或是在构建执行前（`BUILD`）删除构建缓存内容。
    - `clear-configuration-cache-state-before` ：在场景执行前（`SCENARIO`） 、在清理工作执行前（`CLEANUP`）亦或是在构建执行前（`BUILD`）删除 `.gradle/configuration-cache-state` 目录下的内容。
    - `clear-project-cache-before`：在场景执行前（`SCENARIO`） 、在清理工作执行前（`CLEANUP`）亦或是在构建执行前（`BUILD`）删除项目缓存目录 `.gradle` 和 `buildSrc/.gradle` 。

等等，全部配置项可以在[这里](https://github.com/gradle/gradle-profiler#profiling-incremental-builds)找到。下面是一个更加完整的增量构建的 `.scenarios` 文件示例：

```
incremental_build {
    tasks = ["assemble"]

    apply-build-script-change-to = "build.gradle.kts"
    apply-project-dependency-change-to {
        files = ["build.gradle"]
        # Default number of dependency-count is 3.
        # Gradle Profiler will simulate changes to project dependencies by generate some additional projects and then add a combination of project dependencies to every non-generated subprojects before each iteration.
        # The profiler will generate the minimal number of subprojects to allow for a unique combination of dependencies to be used for each iteration.
        # Note: Number of generated projects is calculated as binomial coffiecient: "from `x` choose `dependency-count` = `iterations * files`", where number of generated projects is `x`.
        dependency-count = 3
    }
    apply-abi-change-to = "src/main/java/MyThing.java"
    apply-non-abi-change-to = ["src/main/java/MyThing.java", "src/main/java/MyOtherThing.java"]
    apply-h-change-to = "src/main/headers/app.h"
    apply-cpp-change-to = "src/main/cpp/app.cpp"
    apply-property-resource-change-to = "src/main/resources/thing.properties"
    apply-android-resource-change-to = "src/main/res/values/strings.xml"
    apply-android-resource-value-change-to = "src/main/res/values/strings.xml"
    apply-android-manifest-change-to = "src/main/AndroidManifest.xml"
    clear-build-cache-before = SCENARIO
    clear-transform-cache-before = BUILD
    show-build-cache-size = true
    git-checkout = {
        cleanup = "efb43a1"
        build = "master"
    }
    git-revert = ["efb43a1"]
    jvm-args = ["-Xmx2500m", "-XX:MaxMetaspaceSize=512m"]
}
```

## 分析构建性能

Gradle Profiler 除了可以对构建进行基准测试外，还可以对构建进行更深入的性能分析，找出构建的性能瓶颈。

```bash
gradle-profiler --profile <name-of-profiler> --project-dir <root-dir-of-build> <task>...
```

鉴于本文的主角是基准测试，分析工具这里就不详细展开了。值得一提的是，事实上，Gradle 本身也支持分析构建性能。

```bash
gradlew --profile --offline --rerun-tasks assembleDebug
```

## 参考文档

- Profile your build - [https://developer.android.com/build/profile-your-build](https://developer.android.com/build/profile-your-build)
- Benchmarking Gradle Builds Using Gradle-Profiler - [https://goobar.dev/benchmarking-gradle-builds-using-gradle-profiler/](https://goobar.dev/benchmarking-gradle-builds-using-gradle-profiler/)
- Gradle Profiler GitHub 项目主页 - [https://github.com/gradle/gradle-profiler](https://github.com/gradle/gradle-profiler)