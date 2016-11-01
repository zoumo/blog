---
title: '[Docker1.12源码] Daemon 启动 (1)'
date: 2016-10-25 14:36:09
tags: docker
categories: docker
---

## 前言
Docker在前段时间发布了1.12版本
目前网上看到的源码解析很多都比较旧, 所以我写了这份可能是最新的docker源码解析, 不定期更新
因为是我边看别写的, 所以可能会有些错误的理解, 而且也没有整体架构图和流程图, 后期可能会补上

在Docker1.12.0版本中docker将client和daemon分成两个binary
>Changelog:
Split the binary into two: `docker` (client) and `dockerd` (daemon) [#20639](https://github.com/docker/docker/pull/20639)

两个binary的主入口main函数分别在:
 - docker (client) 位于 `./docker/cmd/docker/docker.go`
 - dockerd (daemon) 位于 `./docker/cmd/dockerd/docker.go`

## 启动流程
Docker Daemon 的main函数很简单

```go
// ./docker/cmd/dockerd/docker.go
func main() {
    if reexec.Init() {
        return
    }

    // Set terminal emulation based on platform as required.
    _, stdout, stderr := term.StdStreams()
    logrus.SetOutput(stderr)

    cmd := newDaemonCommand()
    cmd.SetOutput(stdout)
    if err := cmd.Execute(); err != nil {
        fmt.Fprintf(stderr, "%s\n", err)
        os.Exit(1)
    }
}
```
<!-- more -->

可以看出只做了三件事
1. 执行reexec.Init()
2. 获取到stdout和stderr的文件句柄, 其实就是os.Stdout, os.Stderr, 然后将logrus的日志输出设置为stderr
3. 通过newDaemonCommand()函数建立一个cmd, 然后将cmd的标准输出设置为stdout, 然后执行这个命令, 启动daemon

### 1. 那么`reexec.Init()`是干什么的?

**先说结论, 这个命令是为了让`dockerd`能支持任意的程序调用, 只要在代码中实现了想要的功能**

`docker/pkg/reexec`的 *README.md*
> The `reexec` package facilitates the busybox style reexec of the docker binary that we require because 
of the forking limitations of using Go.  Handlers can be registered with a name and the argv 0 of 
the exec of the binary will be used to find and execute custom init paths.

由于Go在语言层面对派生进程的限制, 这个包实现了类似busybox的程序调用, 即根据argv 0来决定程序的功能

我查找了下资料发现最早是在 [commit 7321067](https://github.com/docker/docker/commit/73210671764fc3de133a627205582e069e1ff43d), docker 1.2 (August 2014):
> This changes the way the exec drivers work by not specifing a -driver flag on reexec.

> For each of the exec drivers they register their own functions that will be matched against the argv 0 on exec and called if they match.
This also allows any functionality to be added to docker so that the binary can be reexec'd and any type of function can be called.
I moved the flag parsing on docker exec to the specific initializers so that the implementations do not bleed into one another.
This also allows for more flexability within reexec initializers to specify their own flags and options.

> Init is called as the first part of the exec process and returns true if an initialization function was called

我们来做个小实验, 看看这个`reexec`包是怎么生效的

```go
// mytest.go
package main

import (
    "fmt"
    "os"

    "github.com/docker/docker/pkg/reexec"
    "github.com/zoumo/logdog"
)

func init() {
    reexec.Register("docker-mountfrom", mountFromMain)
}

func mountFromMain() {
    fmt.Println("this is docker-mountfrom")
}

func main() {
    nArgs := len(os.Args)
    fmt.Println("\nargs : ", os.Args)
    if nArgs == 1 {
        // 检查Init是否成功
        logdog.Info("Init result: ", reexec.Init())

        // 尝试调用docker-mountfrom
        cmd := reexec.Command("docker-mountfrom", "test")
        logdog.Info("Path: ", cmd.Path)
        logdog.Info("Args: ", cmd.Args)
        // 获取到cmd执行结果并打印
        out, err := cmd.CombinedOutput()
        if err != nil {
            fmt.Println(err)
        }
        logdog.Info(string(out))

    } else if nArgs == 2 {
        fmt.Println("Init result: ", reexec.Init())
    }

}

```

输出为([logdog](https://github.com/zoumo/logdog)是我写的一个日志包, 有兴趣可以参考一下)
![docker/reexec_2.png](http://7xjgzy.com1.z0.glb.clouddn.com/docker/reexec_2.png)

可以发现`mytest`运行了两次: 
**第一次: 是我手动调用的`./mytest`**
我们可以看到第一次`reexec.Init()`的输出是`false`, 因为我注册的名字是`docker-mountfrom`, 而`Args[0]`是`./mytest`没有匹配上,所以`Init`没有成功

**第二次: 是`reexec`调用了`/Users/jim/GitHub/go-test/mytest`** 参数`Args`是`docker-mountfrom test`, 
因为`reexec`底层使用`os/exec`来执行外部命令, `exec.Cmd`中要求`Args`中`Args[0]`必须是`command name`, 如果`Args`为空或者`nil`, 则会自动把`Path`填进去,  而这一次`reexec.Init()`结果是`true`

#### 小结 
`reexec.Register()`和`reexec.Init()`是配套使用的, 可以发现docker daemon在启动之前没有任何跟`docker`或者`dockerd`相关名字的`initializer`注册过, 所以`reexec.Init()`肯定返回`false`

`reexec.Command()`以`daemon/graphdriver/overlay2/mount.go`中代码为例:

```go
// ./docker/daemon/graphdriver/overlay2/mount.go
// Init()
reexec.Register("docker-mountfrom", mountFromMain)
...
// mountFrom()
cmd := reexec.Command("docker-mountfrom", dir)
```

可以推测cmd为下面结构

```go
cmd := exec.Cmd {
    Path: "path/to/dockerd",
    Args: []string{"docker-mountfrom", dir}
}
```

这样执行`cmd`之后其实是执行了`path/to/dockerd`的binary, 但是`dockerd`收到的`Args`却是`docker-mountfrom dir`
这样第二次进入了`dockerd`的`main()`函数,  此时`Args[0]`是`docker-mountfrom`, 所以`reexec.Init()`, 则会执行`mountFromMain()`, 我们关注其中两段代码

```go
// ./docker/daemon/graphdriver/overlay2/mount.go

// mountfromMain is the entry-point for docker-mountfrom on re-exec.
func mountFromMain() {
    flag.Parse()
    ...
    if err := os.Chdir(flag.Arg(0)); err != nil {
        fatal(err)
    }
    ...
}
```

其中`flag.Arg(0)`就是`os.Args[1]`即`dir`, `mountFromMain()`完成目录切换 这样就完成了`reexec.Init()`的使命, 返回`true`, `dockerd`整个程序退出!
**Brilliant!!!**, 它是把`dockerd`当做`docker-mountfrom`来用, 但是整个系统里面并没有`docker-mountfrom`这个`binary`, 只要在`dockerd`里面实现了对应的方法, `dockerd`可以成为任意的程序

### 2. 获取标准输入输出的fd, 设置logrus的日志输出

```go
    _, stdout, stderr := term.StdStreams()
    logrus.SetOutput(stderr)
```

我们看到第二段代码中使用了`./docker/pkg/term`包, 这个包的作用是docker自己实现的, 提供了一些与`terminal`相关的方法
与`golang.org/x/crypto/ssh/terminal`有些类似,

### 3. 创建cobra.Command, 并执行
我们来详细探究一下`newDaemonCommand()`函数到底做了什么.
#### 3.1 新建opts - type daemonOptions

```go
// ./docker/cmd/dockerd/docker:newDaemonCommand()
    opts := daemonOptions{
        daemonConfig: daemon.NewConfig(), // *daemon.Config
        common:       cliflags.NewCommonOptions(), // CommonOptions
    }

// 下面是伪代码
// ./docker/daemon/config_xxx.go
// 基本上所有的field都有对应的json tag来支持反序列化, 而且tag与他们在cli中的名字一致
type daemon.Config struct {
    daemon.CommonConfig // 通用跨平台的配置项, ./docker/daemon/config.go
    
    [platform_specific_fields] // 可选的, 各个平台独占的配置项 ./docker/daemon/config_xxx.go
    
}
// ./docker/cli/flags/common.go
// 这个结构体包含了client和daemon在cli中共同用到的配置
type CommonOptions struct {
    Debug      bool
    Hosts      []string
    LogLevel   string
    TLS        bool
    TLSVerify  bool
    TLSOptions *tlsconfig.Options
    TrustKey   string
}
```

这个opts包含两部分, 一个是daemon.Config, 一个是CommonOptions
daemon.Config包含daemon跨平台的配置以及各个操作系统独有的daemon配置CommonConfig
CommonOptions则是client和daemon共有的配置
总结来说, opts分为三块:
1. CommonOptions -- client和daemon共有的配置, 如debug, hosts, tls, log-level等
2. CommonConfig -- daemon独有,但是是跨平台和操作系统的配置
3. platform specific fields -- daemon独有,且不同操作系统独占的配置项, 如Linux和FreeBSD下有cgroup-parent

> 注意: 在CommonConfig中存在有与CommonOptions重复的配置, 如Debug, Hosts, LogLevel, TLS, TLSVerify, 这些配置在daemon start的时候会从CommonOptions拷贝到CommonConfig中

> 哦对了, 这里有个未解决的问题, 当`runtime.GOOS!= linux`的时候, 将`config.V2Only = ture` ?? 是在Linux下面启动v2版本的api?

#### 3.2 建立cobra.Command

```go
//  ./docker/cmd/dockerd/docker:newDaemonCommand()
    cmd := &cobra.Command{
        Use:           "dockerd [OPTIONS]",
        Short:         "A self-sufficient runtime for containers.",
        SilenceUsage:  true,
        SilenceErrors: true,
        Args:          cli.NoArgs,
        RunE: func(cmd *cobra.Command, args []string) error {
            opts.flags = cmd.Flags()
            return runDaemon(opts)
        },
    }
    cli.SetupRootCommand(cmd)
```

这段代码看上去没什么疑问, 但是其中的`Args: cli.NoArgs`引起了我的注意, 我在`github.com/spf13/cobra`的结构体中没有找到这个`field`, 仔细研究了一下, 在`./docker/hack/vendor.sh`中发现了(能看到其他更多的类似用法)

```sh
clone git github.com/spf13/cobra v1.4.1 https://github.com/dnephin/cobra.git
```

当时我的心里出现了万千只草泥马和黑人问号, 果不其然在dnephin/cobra中看到了下面的代码

```go
// dnephin/cobra/args.go
type PositionalArgs func(cmd *Command, args []string) error

// dnephin/cobra/command.go
type Command struct {
    // Expected arguments
    Args PositionalArgs
}
```

> 我去提了一个issue, 作者给我回复了[issue 27415](https://github.com/docker/docker/issues/27415) 
大致意思是, 把代码中的依赖直接改成dnephin/cobra是可行的, 但是他(dnephin)还在等原作者(spf13)接受功能型的PR. 
**原则上Docker团队是希望使用开源社区上游的项目**

另外实际起作用的是`RunE`中定义的函数, 用`runDaemon(opt)`来启动Docker Daemon.
`opts.flags`从`cmd.Flags()`中获取, 而具体`cmd.Flags()`中有哪些东西, 则是第三块代码决定的

#### 3.3 填入flags

```go
//  ./docker/cmd/dockerd/docker:newDaemonCommand()
    flags := cmd.Flags()
    flags.BoolVarP(&opts.version, "version", "v", false, "Print version information and quit")
    flags.StringVar(&opts.configFile, flagDaemonConfigFile, defaultDaemonConfigFile, "Daemon configuration file")
    opts.common.InstallFlags(flags)
    opts.daemonConfig.InstallFlags(flags)
    installServiceFlags(flags)
```

这段代码可以分4个部分来看
第2行: 拿到`flags`, 从3.2中可知, `cmd.Flags()`与`opt.flags`最终是等价的
第3-4行:使`flags`支持两个参数, 分别是`version`和`config-file`
第5行: 使`flags`支持通用的参数, 如debug, log-level, tls, tlsverify, hosts, host等
第6行: 使`flags`支持daemon独有的参数, 如restart, graph等, 以及一些操作系统独有的参数
第7行: 使`flags`支持service相关的参数, window独有, **这里为什么不把这个参数放在上一步做呢, 估计是觉得这几个参数没必要存储在Config里面, 有待考证**

到这里为止, docker daemon已经完成了启动前的准备, 下一步就是`runDaemon()`