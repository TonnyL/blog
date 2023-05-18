# Explore Packman Part 2 - Code Style Check (Spotless + Ktlint)

## 为什么需要代码风格检查

团队协作就意味着需要制定标准，一千个人眼中有一千个哈姆雷特如果发生在团队协作中那极有可能不是一种浪漫，而是一种灾难（BTW “一千个人眼中有一千个哈姆雷特”这句话本身很可能就是一句**山寨版**的英文谚语）。即使独立开发，制定代码风格规范也是很有必要的，统一的代码风格意味着：

- 代码更可读；
- 代码更易维护；
- 更易于代码审查；
- 降低维护成本；
- 减少 bug 产生等。

## 为什么选择 Spotless + Ktlint

[Spotless](https://github.com/diffplug/spotless) 是业内使用非常广泛的代码风格检查和格式化工具，使用过程简单，完全自动化，而且具有以下特点：

1. 支持多种编程语言：C, C++, Java, Kotlin, Python, XML 等等；
2. 插件支持丰富：Gradle, Maven, SBT 。并且支持 IDE 插件如 VS Code, IntelliJ ；
3. 社区活跃。

[Ktlint](https://github.com/pinterest/ktlint) 专门用于检查（lint）和格式化（format） Kotlin 代码的静态代码分析工具。事实上，除了 Ktlint 外，Spotless 还支持 [ktfmt](https://github.com/facebookincubator/ktfmt), [Prettier](https://prettier.io/) 等工具，按需选择即可。

## 配置 Spotless & Ktlint

```kotlin
// build.gradle.kts
allprojects {
		apply<com.diffplug.gradle.spotless.SpotlessPlugin>()
		extensions.configure<com.diffplug.gradle.spotless.SpotlessExtension> {
				format("misc") {
		        target(
		            fileTree(
		                mapOf(
		                    "dir" to ".",
		                    "include" to listOf(
		                        "**/*.md",
		                        "**/.gitignore",
		                        "**/*.yaml",
		                        "**/*.yml",
		                        "**/*.toml",
		                        "**/*.properties",
		                        "**/*.pro"
		                    ),
		                    "exclude" to listOf(
		                        "**/.gradle/**",
		                        "**/.gradle-cache/**",
		                        "**/tools/**",
		                        "**/build/**"
		                    )
		                )
		            )
		        )
		        trimTrailingWhitespace()
		        endWithNewline()
		    }
		
		    format("xml") {
		        target("**/res/**/*.xml")
		        targetExclude("**/build/**")
		        trimTrailingWhitespace()
		        endWithNewline()
		    }
		
		    val editorConfigPath = rootProject.file(".editorconfig")
		    kotlin {
		        target("**/*.kt")
		        ktlint()
		            .setEditorConfigPath(editorConfigPath)
		        trimTrailingWhitespace()
		        endWithNewline()
		    }
		
		    kotlinGradle {
		        target("**/*.kts")
		        ktlint()
		            .setEditorConfigPath(editorConfigPath)
		        trimTrailingWhitespace()
		        endWithNewline()
		    }				
		}
}
```

- `format`、`kotlin`、`kotlinGradle`：对部分语言（例如 Kotlin, Java, C/C++ 等），Spotless 提供对应了配置方法（`kotlin` 和 `kotlinGradle` 等），对于没有直接提供配置方案的语言，我们可以通过 `format` 方法实现；
- `target` 、`targetExclude`：检查的目标文件和需要排除的目标文件；
- `trimTrailingWhitespace` ：去除末尾空格；
- `endWithNewline`：文件以空行结尾。为什么要这样配置呢？简单来说，这是 POSIX 标准定义的规则：
    
    > **3.206 Line**
    > 
    > 
    > A sequence of zero or more non- <newline> characters plus a terminating <newline> character.
    > 
    
    更加详细的解释可以参考  [https://stackoverflow.com/a/729795/5835014](https://stackoverflow.com/a/729795/5835014) 。
    
- `ktlint` ：对 Ktlint 的配置。
- 除了这些配置项外，还有 `licenseHeader` 配置文件的许可证声明、`removeUnusedImports` 去除无用导包等等，详细列表参见 [https://github.com/diffplug/spotless/tree/main/plugin-gradle#quickstart](https://github.com/diffplug/spotless/tree/main/plugin-gradle#quickstart) 。

## .editorconfig

.editorconfig 文件是一种通用的编辑器和 IDE 配置文件，它定义了统一的编码风格，几乎所有的主流编辑器都内建或通过插件支持了对 .editorconfig 文件的解析和支持。

```python
# https://editorconfig.org
root = true # 是否为顶级配置文件，true 表示停止搜索 .editorconfig 文件

[*]
indent_style = space # 缩进风格，tab 或者 space
indent_size = 2 # 缩进大小

end_of_line = lf # 换行符类型，换行 lf, 回车 cr，回车换行 crlf
charset = utf-8 # 字符编码类型
trim_trailing_whitespace = true # 是否删除行末尾空格
insert_final_newline = true # 文件末尾是否插入空行

[*.{java,kt,kts,scala,rs,xml,kt.spec,kts.spec}] # 特定文件的覆盖规则
indent_size = 4

[*.{kt,kts}]
# ij_* 为 intellij idea 的规则
ij_kotlin_allow_trailing_comma = false
ij_kotlin_allow_trailing_comma_on_call_site = false

# ktlint_* 为 ktlint 的配置规则
ktlint_standard = enabled
ktlint_experimental = enabled
ktlint_code_style = official

ktlint_standard_no-unused-imports = disabled
ktlint_standard_no-wildcard-imports = disabled
ktlint_standard_no-unused-imports = disabled
ktlint_standard_no-blank-line-before-rbrace = disabled
ktlint_standard_chain-wrapping = disabled
ktlint_standard_no-empty-first-line-in-method-block = disabled
ktlint_standard_trailing-comma-on-call-site = disabled
ktlint_standard_trailing-comma-on-declaration-site = disabled

[*.md]
trim_trailing_whitespace = true
```

Ktlint 的配置也是通过 .editorconfig 文件完成的（调用 `setEditorConfigPath` 方法）。

- `ktlint_standard` ：是否启用 Ktlint 定义的[标准规则](https://pinterest.github.io/ktlint/rules/standard/)；
- `ktlint_experimental`：是否启用 Ktlint 定义的[试验规则](https://pinterest.github.io/ktlint/rules/experimental/)；
- `ktlint_code_style`：代码风格偏好，支持
    - `intellij_idea` ：尽量和 Intellij IDEA 默认 formatter 的风格保持一致，基于  [Kotlin 编码规范](https://kotlinlang.org/docs/coding-conventions.html)；
    - `android_studio` ：尽量和 Android Studio 默认 formatter 的风格保持一致，基于 [Android Kotlin 风格指南](https://developer.android.com/kotlin/style-guide)；
    - `ktlint_official` ：集合了 [Kotlin 编码规范](https://kotlinlang.org/docs/coding-conventions.html) 和 [Android Kotlin 风格指南](https://developer.android.com/kotlin/style-guide) 的**部分**规则，注意应用了这种风格后，与 Intellij IDEA 和 Android Studio 的 formatter 都不兼容了，所以最好是禁用 IDE 的重排版功能。
- `ktlint_standard_*`：是否启用特定规则。

插件定义好并成功应用之后，可以运行

- `./gradlew spotlessCheck` 任务检查项目内的缩进、换行等格式问题，如果有问题则任务会失败，即 lint；
- `./gradlew spotlessApply` 任务在 `./gradlew spotlessCheck` 任务的基础上，会自动修改有问题的代码，即 format 。

运行这些 Gradle 任务的合适的时机是什么呢？

1. 绑定 build 任务，但是 build 是非常高频运行的任务，会造成非常可观的不必要的耗时；
2. 作为 CI 任务，缺点是需要推送代码之后才能知道结果；
3. 还有一种可能便是 Git Hooks。

## Git Hooks

Git Hooks 即 Git 钩子，是能在特定动作触发时的自定义脚本。Git Hooks 分为两种：

- 客户端钩子：用于由诸如提交、合并等操作；
- 服务端钩子：用于接收被推送的提交等联网操作。

Git Hooks 位于项目中的 `.git/hooks` 目录下，初始情况下由以下示例组成：

```bash
applypatch-msg.sample
commit-msg.sample
post-update.sample
pre-applypatch.sample
pre-commit.sample
prepare-commit-msg.sample
pre-push.sample
pre-rebase.sample
pre-receive.sample
update.sample
```

这些示例脚本均为 shell 脚本，其中一些还混杂了 Perl 代码。不过，我们可以用 Python 、Ruby 或者任何语言编写。去掉 `.sample` 后缀即可启用 hook 。

关于这些 hook 的类型及其作用，可以访问 [https://git-scm.com/docs/githooks](https://links.jianshu.com/go?to=https%3A%2F%2Fgit-scm.com%2Fdocs%2Fgithooks)。我们想要在工程师推送本地改动到远端时，检查代码风格，如果未通过检查则禁止推送本地改动，这需要用到的是 **pre-push** :

```bash
#!/bin/sh

echo "Running git pre-push hook"
# If .gitmodules file exists, it's in the root git repository now.
if [ -e .gitmodules ]; then
    ./gradlew spotlessCheck
else
    .././gradlew spotlessCheck
fi

# return 1 exit code if running code style checks fails.
if [ $? -ne 0 ]; then
    echo "Code style does not match defined rules, see output details and run ./gradlew spotlessApply if needed."
    exit 1
fi

exit 0
```

但是，这种修改方式只能在本地机器生效，其他工程师是没有共享此修改的，因为 `.git` 目录是不能随着提交一起推送到远端的。我们使用的方案是：将 hook 脚本放置在 `.git` 目录以外的地方，每次 gradle 运行任务时复制 hook 脚本至 `.git/hooks` 目录下。

```kotlin
// build.gradle.kts
allprojects {
    val copyHooksScriptPath = "./scripts/copyHooks.sh"
    afterEvaluate {
        if (rootProject.file(copyHooksScriptPath).exists()) {
            println("copy git hook files")

            runCatching {
                Runtime.getRuntime().exec(
                    if (isWindows) {
                        "PowerShell $copyHooksScriptPath"
                    } else {
                        copyHooksScriptPath
                    }
                )
            }
        }
    }
}
```

```bash
#!/bin/bash
declare executableFileFolder="./scripts/hooks"
executableFiles=()
for executableFile in "$executableFileFolder"/*; do
  executableFiles+=("${executableFile}")
  echo "executable hook file found: $executableFile"
done

executableFilesCount=${#executableFiles[@]}
if ((${#executableFilesCount[@]} == 0)); then
  echo "no executable hook file found!"
  # no hooks defined.
  exit 0
fi

targetHooksDir="./.git/hooks"

# Delete existing hook files if needed.
rm -rf "$targetHooksDir/*"

for customizedScriptPath in "${executableFiles[@]}"; do
  # Copy executable file to gitRepositories' hooks folder.
  cp "$customizedScriptPath" "$targetHooksDir"
done
```

在某些紧急情况下，我们可能就是想要跳过 Git Hooks 检查（我十分非常不建议这样做，而且请务必了解这样做所带来的后果），我们可以直接 Git 命令之后增加 `--no-verify` 参数，例如 `git push origin/main --no-verify` 。在 Android Studio 或 Intellij IDEA 的 Git 提交面板中，取消勾选 **Run Git hooks** 。

另外，Ktlint 官方提供了一键安装 Git Hooks 的方法，可参考：[https://pinterest.github.io/ktlint/install/cli/#git-hooks](https://pinterest.github.io/ktlint/install/cli/#git-hooks) 。

## **Ktlint 常见报错信息及修改方法**

有一些错误是 `spotlessCheck` 可以检查出，但是 `spotlessApply` 不能自动修改的。以下是一些常见错误提示：

- `File 'MainActivity.kt' contains a single class and possibly also extension functions for that class and should be named same after that class 'MainActivity.kt'`：一个 kt 文件只包含了一个类，而这个类名和 kt 文件名不相同。将 kt 文件名改为和类名相同即可。
- `File name 'utils.kt' should conform PascalCase`：文件名应该遵循大驼峰命名规则。将 utils.kt 改为 Utils.kt 即可。
- `Argument should be on a separate line (unless all arguments can fit a single line)`：将参数列表分行排布即可。