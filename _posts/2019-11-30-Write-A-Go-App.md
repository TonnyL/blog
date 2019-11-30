---
layout: post
title: 写一个 Golang cli 程序吧
summary: 本文会介绍如何编写一个 go 命令行(cli)程序, 用于浏览 GitHub trending repositories 和 developers.
featured-img: wukong
categories: [Go]
---

# 写一个 Golang cli 程序吧
本文会介绍如何编写一个 go 命令行(cli)程序, 用于浏览 GitHub trending repositories 和 developers.
首先我们看一下最终的实现效果:

```shell
% wukong repo -lang go -period daily

+------+----------------------------+--------------------------+----------+---------------------+-------------------------------------------------------------------+
| RANK |            NAME            |       DESCRIPTION        | LANGUAGE | STARS(TOTAL/PERIOD) |                                URL                                |
+------+----------------------------+--------------------------+----------+---------------------+-------------------------------------------------------------------+
|    1 | OpenDiablo2                | An open source re-implem | Go       | 2626/1625           | https://github.com/OpenDiablo2/OpenDiablo2                        |
|      |                            | entation of Diablo 2     |          |                     |                                                                   |
+------+----------------------------+--------------------------+----------+---------------------+-------------------------------------------------------------------+
|    2 | validator                  | 💯Go Struct and Field    | Go       | 4154/149            | https://github.com/go-playground/validator                        |
|      |                            | validation, including Cr |          |                     |                                                                   |
|      |                            | oss Field, Cross Struct, |          |                     |                                                                   |
|      |                            |  Map, Slice and Array di |          |                     |                                                                   |
|      |                            | ving                     |          |                     |                                                                   |
+------+----------------------------+--------------------------+----------+---------------------+-------------------------------------------------------------------+
|    4 | grpc-go                    | The Go language implemen | Go       | 10019/58            | https://github.com/grpc/grpc-go                                   |
|      |                            | tation of gRPC. HTTP/2 b |          |                     |                                                                   |
|      |                            | ased RPC                 |          |                     |                                                                   |
+------+----------------------------+--------------------------+----------+---------------------+-------------------------------------------------------------------+
|    . | ...                        | ...                      | ...      |  ...                | ...                                                               |
+------+----------------------------+--------------------------+----------+---------------------+-------------------------------------------------------------------+
```

```shell
% wukong dev

+------+------------------------------+--------------------------+----------------------------------+
| RANK |             NAME             |  REPO NAME/DESCRIPTION   |               URL                |
+------+------------------------------+--------------------------+----------------------------------+
|    1 | Alon Zakai(kripken)          | sql.js - SQLite compiled | https://github.com/kripken       |
|      |                              |  to JavaScript through E |                                  |
|      |                              | mscripten                |                                  |
+------+------------------------------+--------------------------+----------------------------------+
|    2 | Klaus Post(klauspost)        | compress - Optimized com | https://github.com/klauspost     |
|      |                              | pression packages        |                                  |
+------+------------------------------+--------------------------+----------------------------------+
|    3 | siddontang(siddontang)       | ledisdb - a high perform | https://github.com/siddontang    |
|      |                              | ance NoSQL powered by Go |                                  |
+------+------------------------------+--------------------------+----------------------------------+
|    . | ...                          | ...                      | ...                              |
+------+------------------------------+--------------------------+----------------------------------+
```

## 编码
### API
GitHub 官方并没有提供 trending 相关的 API, 不过 https://github.com/huchenme/github-trending-api 提供了一个非官方的实现, 我们可以通过它实现我们的需求.

github-trending-api 提供了3个 API:
+ https://github-trending-api.now.sh/repositories?language=javascript&since=weekly: 用于查询 trending repositories, 其中 `language` 为可选参数, 默认为 空字符串(""), 所有侯选值可以通过第三个 API 获取, `since` 也为可选参数, 默认为 `daily`, 候选值为 `daily`, `weekly`, `monthly`;
+ https://github-trending-api.now.sh/developers?language=javascript&since=weekly: 用于查询 trending developers, 参数类型和含义上一个 API 类似;
+ https://github-trending-api.now.sh/languages: 获取上面的接口中 `language` 参数的所有候选值.

在调用第一个和第二个 API 时, 我们都要调用到第三个 API, 因为我们需要验证 `language` 参数的合法性, 这可能会给 `github-trending-api` 造成很多不必要的请求而增加原作者的服务器的压力, 并且理论上说 languages 数据的变化不会很频繁, 所以我们直接把 languages 数据直接放在 GitHub 上, 每次请求从 GitHub 的服务器请求即可. 最终第三个 API 的 URL 就被我们替换为了 https://raw.githubusercontent.com/TonnyL/Wukong/master/resources/languages.json .

### data struct
以 `https://github-trending-api.now.sh/repositories?language=javascript&since=weekly` 这个接口为例, 返回的 JSON 格式的数据为:
```json
[
  ...
  {
    "author": "google",
    "name": "gvisor",
    "avatar": "https://github.com/google.png",
    "url": "https://github.com/google/gvisor",
    "description": "Container Runtime Sandbox",
    "language": "Go",
    "languageColor": "#3572A5",
    "stars": 3320,
    "forks": 118,
    "currentPeriodStars": 1624,
    "builtBy": [
      {
        "href": "https://github.com/viatsko",
        "avatar": "https://avatars0.githubusercontent.com/u/376065",
        "username": "viatsko"
      }
    ]
  }
  ...
]
```

对应的 data struct 即为:

```go
type Repository struct {
    // google
    Author string `json:"author"`
    // gvisor
    Name string `json:"name"`
    // https://github.com/google.png
    Avatar string `json:"avatar"`
    // https://github.com/google/gvisor
    Url string `json:"url"`
    // Container Runtime Sandbox
    Description string `json:"description"`
    // Go
    Language string `json:"language"`
    // #3572A5
    LanguageColor string `json:"languageColor"`
    // 3320
    Stars int32 `json:"stars"`
    // 118
    Forks int32 `json:"forks"`
    // 1624
    CurrentPeriodStars int32     `json:"currentPeriodStars"`
    BuiltBy            []BuiltBy `json:"builtBy"`
}
```

比较简单, 这里不再解释了.

### request
发起一个 `get` 类型的 http 请求即可. 以获取 trending repositories 为例:

```go
// Receive an array of trending repositories.
func FetchTrendingRepositories(language, since string) ([]Repository, error) {
    resp, err := http.Get(fmt.Sprintf("https://github-trending-api.now.sh/repositories?language=%s&since=%s", language, since))
    if err != nil {
        return nil, err
    }

    defer resp.Body.Close()

    repositories := make([]Repository, 0)
    jsonErr := json.NewDecoder(resp.Body).Decode(&repositories)
    if jsonErr != nil {
        return nil, jsonErr
    }

    return repositories, nil
}
```

### table writer
获取到数据之后, 要这样展示给用户呢? https://github.com/olekukonko/tablewriter , 用 ASCII 表格的形式. `tablewriter` 提供了丰富的功能, 我们以展示 trending repositories 为例:
```go
func ShowTableOfRepositories(repos []Repository) {
    table := tablewriter.NewWriter(os.Stdout)
    table.SetHeader([]string{"Rank", "Name", "Description", "Language", "Stars(Total/Period)", "Url"})
    table.SetRowLine(true)

    for index, repo := range repos {
        table.Append([]string{strconv.Itoa(index + 1), repo.Name, repo.DisplayDescription(), repo.DisplayLanguage(), fmt.Sprintf("%d/%d", repo.Stars, repo.CurrentPeriodStars), repo.Url})
    }

    table.Render()
}
```

`table.SetHeader([]string{"Rank", "Name", "Description", "Language", "Stars(Total/Period)", "Url"})` 设置表格的头, `table.SetRowLine(true)` 设置显示单元格的边界线, 然后遍历一下 slice 中的数据, 每一条数据作为一行展示 `table.Append([]string{strconv.Itoa(index + 1), repo.Name, repo.DisplayDescription(), repo.DisplayLanguage(), fmt.Sprintf("%d/%d", repo.Stars, repo.CurrentPeriodStars), repo.Url})` .

### cli
go 官方提供了一个名为 flag 的包, 方便进行命令行解析. 不过我们使用的是 https://github.com/urfave/cli 这个第三方库, 因为使用起来比较简单, 功能也比较丰富. 还有一个比较出名的 [cobra](https://github.com/spf13/cobra), 功能更多, 很强大, 相应地使用起来也更复杂. 我们使用 cli 即可.

```go
func main() {
    var lang string
    var period string

    flags := []cli.Flag{
        cli.StringFlag{
            Name:        "lang,l",
            Usage:       "language, default to `all`, use list command to see all the available options",
            Required:    false,
            Destination: &lang,
        },
        cli.StringFlag{
            Name:        "period,p",
            Usage:       "Period, default to `daily`, possible values: daily, weekly and monthly",
            Required:    false,
            Destination: &period,
        },
    }

    app := cli.NewApp()
    app.Name = "Wukong" // Incredible name!
    app.Usage = "A command-line tool for browsing GitHub trending repositories&developers written by Go."
    app.Version = "0.1.0-alpha02"
    app.Copyright = "Wukong is under an MIT license. See the [LICENSE](https://github.com/TonnyL/Wukong/blob/master/LICENSE) for more information."
    app.Commands = []cli.Command{
        {
            Name:        "repo",
            Aliases:     []string{"r", "repositories", "repository"},
            Description: "See the developers that the GitHub community is most excited about.",
            Usage:       "-lang x -period y",
            Flags:       flags,
            Action: func(c *cli.Context) error {
                params := CheckParams(lang, period)
                if params == nil {
                    return errors.New("🧐 Invalid params")
                }

                repos, err := FetchTrendingRepositories(params.Lang, params.Period)
                if err != nil {
                    return err
                }

                ShowTableOfRepositories(repos)

                return nil
            },
        },
        {
            Name:        "dev",
            Aliases:     []string{"d", "developers", "developer"},
            Description: "See the repositories that the GitHub community is most excited about.",
            Usage:       "-lang x -period y",
            Flags:       flags,
            Action: func(c *cli.Context) error {
                params := CheckParams(lang, period)
                if params == nil {
                    return errors.New("🧐 Invalid params")
                }

                devs, err := FetchTrendingDevelopers(params.Lang, params.Period)
                if err != nil {
                    return err
                }

                ShowTableOfDevelopers(devs)

                return nil
            },
        },
        {
            Name:    "lang",
            Aliases: []string{"l", "languages", "language"},
            Usage:   "List all the available language options",
            Action: func(c *cli.Context) error {
                langs, err := FetchLanguages()
                if err != nil {
                    return err
                }

                ShowTableOfLanguages(langs)

                return nil
            },
        },
    }
    err := app.Run(os.Args)
    if err != nil {
        fmt.Print("☹️ command error: " + err.Error())
    }
}
```

- flags 即为用户可以输入的参数, 我们定义了 `lang` 和 `period` 两个 string 类型变量并绑定对应的 flag;
- app.Name 指定我们的 cli 应用的名称;
- app.Usage 描述我们的应用的用途是什么;
- app.Version 描述我们的应用的版本;
- app.Copyright 描述我们的应用的版权信息;
- app.Commands 为关键部分, 定义了三个可以执行的命令: `repo`, `dev` 和 `lang` . `repo` 列出 trending repositories, `dev` 列出 trending developers, `lang` 列出所有支持的编程语言.
    + Name: 命令的名称;
    + Aliases: 命令的别名, 一个命令可以有多个别名;
    + Description: 命令的描述信息;
    + Usage: 命令的用法;
    + Flags: 命令接收的参数;
    + Action: 命令如何执行.
    
我们还需要验证一下用户输入参数的合法性, 实现默认参数以及减少不必要的请求.

```go
func CheckParams(lang, period string) *Params {
    if period == "" {
        period = "daily"
    }

    if period != "daily" && period != "weekly" && period != "monthly" {
        return nil
    }

    if lang == "" || lang == "all" {
        return &Params{
            Lang:   "",
            Period: period,
        }
    } else {
        languages, err := FetchLanguages()
        if err != nil {
            return nil
        }

        for _, l := range languages {
            if l.UrlParam == lang {
                return &Params{
                    Lang:   lang,
                    Period: period,
                }
            }
        }
    }

    return nil
}
```

## 编译为可执行程序
`go run` 可以让编写的 go 程序解释执行. 在对应目录执行 `go run`  即可:

```shell
go run data.go main.go
```

`go build` 可以编译成对应平台的可执行程序. 在对应目录执行 `go build`:

```shell
go build -o wukong data.go main.go
```

执行完毕后, 会在当前目录下生成一个名为 `wukong` 的可执行文件, 双击该文件即可执行.

上面的命令只会生成当前系统对应的可执行文件, 那如果我在 macOS 上, 想要生成 Linux 平台的可执行文件呢? 这个时候就要用到交叉编译了.
在 macOS 或 Linux 上编译其他平台的二进制可执行文件:

```shell
# macOS
CGO_ENABLED=0 GOOS=darwin GOARCH=amd64 go build ...
# Linux
CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build ...
# Windows
CGO_ENABLED=0 GOOS=windows GOARCH=amd64 go build ...
```

在 Windows 上编译其他平台的二进制可执行文件:

```shell
# macOS
CGO_ENABLED=0 GOOS=darwin GOARCH=amd64 go build server.go
# Linux
CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build server.go
# Windows
CGO_ENABLED=0 GOOS=windows GOARCH=amd64 go build server.go
```

其中,参数 `GOOS`, `GOARCH`, `CGO` 参数的含义分别是:
+ GOOS: 目标平台的操作系统(darwin, freebsd, linux, windows 等)
+ GOARCH: 目标平台的 CPU 体系架构(386, amd64, arm 等. 使用 `go tool dist list` 查看支持的 OS 和架构列表.)
+ CGO_ENABLED: 设置为0表示不启用CGO进行编译, 代码中如果有用到C语言相关的库,编译时就需要开启 `CGO_ENABLED=1`

## 压缩可执行文件
首先我们看一下 `go build` 生成的可执行文件的大小:

```shell
% ls -lh

total 20200
-rw-r--r--  1 tonnyl  staff   2.5K Nov 19 00:50 data.go
-rw-r--r--  1 tonnyl  staff   1.8K Nov 17 18:05 data_test.go
-rw-r--r--  1 tonnyl  staff   5.4K Nov 19 01:19 main.go
-rw-r--r--  1 tonnyl  staff   1.5K Nov 17 18:05 main_test.go
-rwxr-xr-x@ 1 tonnyl  staff   9.8M Nov 17 20:53 wukong
```
9.8M, 你没有看错. 事实上, 就算是最简单的 hello world 变异后也有 2M 之巨. 我们用到了一些第三方库, 体积就来到了近 10M 之多. 
相比较而言, C 语言版的 hello world 编译出来的可执行文件仅是 KB 级别. 

为什么体积会这么大呢? 这里是官方的回答: https://golang.org/doc/faq#Why_is_my_trivial_program_such_a_large_binary . 简单来说就是: go 使用的是静态编译, 打包出来的可执行文件包含了 go 运行时, 而 C 是以动态链接库的形式编译的. 更多关于 go 和 C 语言编译的差异, 可以访问 https://www.cnxct.com/why-golang-elf-binary-file-is-large-than-c/ .

如果减小生成的可执行文件的体积呢?
1. 设置编译参数 `-ldflags`:
    设置编译参数 `-ldflags "-w -s"`, 其中 `-w` 去掉调试信息(无法使用gdb调试), `-s` 去掉符号信息.

    ```shell
    % go build -o wukong -ldflags "-s -w" main.go data.go
    % ls -lh                                             
    total 16176
    -rw-r--r--  1 tonnyl  staff   2.5K Nov 19 00:50 data.go
    -rw-r--r--  1 tonnyl  staff   1.8K Nov 17 18:05 data_test.go
    -rw-r--r--  1 tonnyl  staff   5.4K Nov 19 01:19 main.go
    -rw-r--r--  1 tonnyl  staff   1.5K Nov 17 18:05 main_test.go
    -rwxr-xr-x  1 tonnyl  staff   7.9M Nov 17 21:57 wukong 
    ```

    体积来到了 7.9M, 减少了约 20%.

2. 使用 UPX 压缩
    [UPX](https://upx.github.io/) 是一款用于压缩可执行程序文件的压缩器. 以下以 macOS 为例(其他平台也支持, 仅安装和部分使用命令不同):
    
    ```shell
    % brew install upx
    % upx wukong
    % ls -lh    
    total 7672
    -rw-r--r--  1 tonnyl  staff   2.5K Nov 19 00:50 data.go
    -rw-r--r--  1 tonnyl  staff   1.8K Nov 17 18:05 data_test.go
    -rw-r--r--  1 tonnyl  staff   5.4K Nov 19 01:19 main.go
    -rw-r--r--  1 tonnyl  staff   1.5K Nov 17 18:05 main_test.go
    -rwxr-xr-x  1 tonnyl  staff   3.0M Nov 17 21:57 wukong 
    ```

    体积来到了 3M, 相比最初的 9.8M 减少了近 70%, 效果显著.

## 发布到 Homebrew
Homebrew 是一个流行的软件包管理器, 支持了 macOS 和 Linux, 得益于 [Windows Subsystem for Linux (WSL)](https://docs.microsoft.com/en-us/windows/wsl/about), Homebrew 也间接支持了 Windows.
brew 有啤酒🍺的意思, homebrew 自然就是「家酿」了, 这有助于我们理解 Homebrew 的一些概念:

+ Keg(酒桶): 用户安装好的脚本, 软件等;
+ Cellar(酒窖): 所有用 Homebrew 安装在本地的脚本或软件组成的集合;
+ Formula(配方): 定义如何下载, 编译和安装脚本或软件的 Ruby 脚本;
+ Tap: 软件仓库.

当我们输入 `brew install go`, 发生了什么呢? Homebrew 首先会在 Tap 中寻找 `go` 对应的 Ruby 脚本(Tap 中存放的并不是编译好的二进制文件, 而是 Formula), 然后按照脚本中定义的方法将 Keg 下载, 编译, 安装到 Cellar 中.

Homebrew 提供了两个名为 `homebrew/homebrew-core` 和 `homebrew/homebrew-cask` 的官方 Tap, `homebrew-core` 存放了一些流行的 Formula, 如果你的应用满足了[这些需求](https://github.com/Homebrew/brew/blob/master/docs/Versions.md)之后, 也可以向官方申请以收录到 `homebrew-core` 中; 而  `homebrew-cask` 是 Homebrew 的一个扩展, 用于安装 macOS 原生应用, 所以不在我们的讨论范围. 

了解了 Homebrew 的工作流程之后, 想要让我们的应用支持 Homebrew 安装, 大致流程为:
1. 创建我们自己的 Tap(Git 仓库), 用于存放 Formula;
2. 创建 Formula, 用 Ruby 脚本描述我们的应用如果被下载, 安装等.

### 创建 Tap
Tap 其实是一个 Git 仓库, 所以我们直接在 GitHub 上创建一个仓库即可. 但是仓库名有一定的要求 -- 必须是 `homebrew-foobar` 的形式. 例如我的 Tap 仓库的名字为 [`homebrew-tap`](https://github.com/TonnyL/homebrew-tap) .

在本地 `homebrew-tap` 仓库根目录执行以下命令让 Homebrew 可以跟踪我们的 Tap:

```shell
% brew tap Your-GitHub-Username/taps
```
或

```shell
% brew tap github.com/Your-GitHub-Username/homebrew-taps
```

### 创建 Formula
生成二进制可执行文件压缩包的哈希文件, 后面会用到.

```shell
% tar -czf wukong.tar.gz wukong
% shasum -a 256 wukong.tar.gz > wukong.tar.gz.sha256
```

我们首先把 wukong 的源码发布一个 release, 并且附上之前生成的二进制可执行文件压缩包和其对应的哈希文件. release 发布成功后, 复制 `wukong-macos.tar.gz` 的链接地址. 然后执行命令:

```shell
% brew create https://github.com/TonnyL/Wukong/releases/download/v0.1.0-alpha02/wukong-macos.tar.gz
```

Homebrew 会在 `/usr/local/Library/Formula/` 目录下创建一个名为 `filename.rb` 的文件, 其文件名取决于上面链接地址指向的文件的名称. 复制这个文件到本地 `homebrew-tap` 仓库中 `Formula` 目录下, 更名为 `wukong.rb` 并打开:

```ruby
class Wukong < Formula
  desc "A command-line tool for browsing GitHub trending repositories and developers written by Go."
  homepage "https://github.com/TonnyL/Wukong/"
  version "v0.1.0-alpha02"

  bottle :unneeded

  if OS.mac?
    url "https://github.com/TonnyL/Wukong/releases/download/v0.1.0-alpha02/wukong-macos.tar.gz"
    sha256 "446d2cd7f185778b020ae55f721c7b2edb4bf4b919706733ab8f3d3d9188e523"
  elsif OS.linux?
    url "https://github.com/TonnyL/Wukong/releases/download/v0.1.0-alpha02/wukong-linux.tar.gz"
    sha256 "cc4df15ab5e3cf757ae9aa79c2d9d6793947cf4a27cc219ca6c1c6f0c9d6c2b9"
  end

  def install
    bin.install "wukong"
  end

  test do
     system "#{bin}/wukong", "--help"
  end
end

```

desc 即简介, description; homepage 即主页地址; version 即应用版本; url 和 sha256 即二进制可执行文件的地址及其哈希值. 其中 url 和 sha256 会根据 OS 的不同赋予不同的值. 
默认情况下, Homebrew 会编译源代码, 但是我们现在讨论的是预编译版本, 所以不用考虑编译过程, 直接调用 Homebrew 提供的 `bin.install` 即可将预编译好的二进制文件安装到 `/usr/local/bin/`. 
有时候安装成功之后, 功能不一定完备. 我们也可以写测试用例验证功能是否完整. 这里我们简单输出软件版本, 不做深入讨论.
Bottle 也不做讨论.

保存之后我们即可开始安装:

```shell
% brew install TonnyL/tap/Wukong
==> Installing wukong from tonnyl/tap
==> Downloading https://github.com/TonnyL/Wukong/releases/download/v0.1.0-alpha0
Already downloaded: /Users/tonnyl/Library/Caches/Homebrew/downloads/f97e0d8bde2c8dcaf77d5900cd6445b33069f91e51fb59d3f7aa07fdd264b758--wukong-macos.tar.gz
🍺  /usr/local/Cellar/wukong/v0.1.0-alpha02: 3 files, 3.0MB, built in 6 seconds 
```

## 总结
本文源代码: https://github.com/TonnyL/Wukong
