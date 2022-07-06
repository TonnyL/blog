# Android Gradle 构建速度为什么这么慢，还有救吗？

## 为什么
### Gradle 本身就很慢
Gradle 需要运行在 JVM 环境中，所以可以注意到，当一个 Gradle 任务运行起来时，Java 也被启动了。而 Java 就以内存和 CPU 占用大户而远近闻名，因为 Java 需要先启动一个虚拟机，而这个虚拟机的冷启动是非常“重”的，但是一旦可以热启动之后，情况会有很大改善。

### 你在写代码
上班期间，如果你在摸鱼的间隙还写了几行项目的代码，代码行数增加了，那不可避免地构建时间会增加。不仅你在写代码，你的同事也在写代码，而且团队成员的数量也在增加（其实也不一定，团队成员可能被[毕业](https://www.zhihu.com/question/525382704)了）。

### 大家都在写代码
1. 你的项目可能引入一些第三方库或者自定义 Gradle 插件，这些插件在开发时，可能并没有遵守最佳实践，本身就很慢；
1. 防不胜防的因素，比如2017年的 [Spectre 和 Meltdown](https://www.cloudflare.com/zh-cn/learning/security/threats/meltdown-spectre/) 补丁对新的进程和 I/O 性能的影响，直接导致全量构建慢了 50% 到 140%。

### 硬件落伍了
1. 你的电脑可能本身配置就太低，你电脑的配置，如果和你大学机房或者你妈办公室的电脑不分伯仲的话，那就是耶稣也救不了你的构建速度，张家辉说的；
2. 你的电脑和你一样，本身也在老化：风扇不断积灰，硬盘读写速度下降，其他软件占用的 CPU 和运行内存也越来越多（点名 [electron](https://www.electronjs.org/)），系统本身性能也在改变（[Tim Apple 按下了他桌子上的按钮，你的电脑就变慢了](https://www.pingwest.com/a/148190)）等等。所有这些因素加起来，让本不富裕的 Android Studio、Gradle、Java 的算力雪上加霜，构建时间自然而然地增加了。

## 怎么办
### 监控
既然构建速度已经这么慢了，可不能再让他继续变得更慢了，至少不要下降地太快，所以监控是非常有必要的。在新代码被合入主干之前，应该做自动化检查，确认没有大幅度影响构建性能（和产生其他问题）之后再被合入。如果 tradeoff 是值得的，应该在经过讨论之后再决定是否合入。

### 软件层面
1. 先得弄清楚，构建过程的瓶颈在哪里，有的放矢。Gradle 提供了[分析构建过程](https://developer.android.com/studio/build/profile-your-build)的功能，可以直观展示每个任务的耗时是多少；
1. 及时升级 Gradle 及插件版本：Gradle 团队本身也在不断地对 Gradle 进行优化，尝试减少 Gradle 构建速度，例如引入了文件系统监控和[配置缓存](https://blog.Gradle.org/introducing-configuration-caching)等，大幅提升没戏，小幅度提升还是有的；
1. 将项目模块化，这样 Gradle 就会只编译你修改的模块，这项工作费时费力，但是一定是值得的。
1. 保证你的网络能顺利连接互联网，不仅仅是局域网，Gradle 需要联网下载一些依赖，当然你也可以使用国内的镜像。

### 硬件层面
这里引用一句马化腾的话再合适不过了：充钱才能变强（我不为马化腾真的说过这句话负责）。换句话说，买新电脑或者升级配置。Android 构建是 CPU 和 I/O 敏感的，性价比的比较高的方案是升级 CPU、 SSD 和运行内存。不要在你吃饭的家伙事儿上省钱。

拥有聪明小脑瓜的你可能又想到了，既然是 CPU 敏感的，那我可不可以升级一下显卡，用老黄 3090 核弹 GPU 加加速，给 CPU 减轻一下压力呢？答案是不太行。两个方面的原因：

1. 首先你从海鲜二手市场收来的女生自用一手核弹可能是矿渣，没整几天就废了；
1. 更重要的，GPU 还是用来打游戏或者炼丹吧，它不适合做构建这样的任务。虽然有配合 GCC 和 LLVM 实现 GPU 加速的（CUDA）案例，但是对于其他更加通用的编译仍然不适合，更多信息可以参考[这里](https://stackoverflow.com/questions/8417053/is-it-possible-to-use-gpu-acceleration-on-compiling-multiple-programs-on-a-gcc-c)。

### 另辟蹊径
1. 事实上，除了 Gradle，构建 Android 项目还有其他的方式，比如 [Bazel](https://developer.android.com/codelabs/bazel-android-intro#0)、[Buck](https://buck.build/) 等，各有优劣，因为他们并不是 Google Android 团队的官方解决方案，这里就不延伸了；
1. Windows 老先生升级过程中说的一句话非常好，送给大家：坐和放宽。隔壁 Xcode 构建速度也没快到哪里去，都在摆烂呢。

## 结论
润去别的赛道，or 世界加钱可及。