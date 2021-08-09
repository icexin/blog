---
title: "将Go程序跑在裸机上之LibOS"
date: 2021-08-08T01:05:45+08:00
draft: false
---

# 前言

之前我写过一篇探索在裸机上运行 Go 程序的[文章](https://zhuanlan.zhihu.com/p/265806072)，里面介绍了在裸机上运行 Go 代码需要的技术，顺便介绍了 eggos，一个运行于裸机的 unikernel。

在编写[eggos](https://github.com/icexin/eggos)的过程中，我发现刨去内核代码，主程序代码其实就是一个普通的 Go 程序，跟其他 Go 程序一样从 `main` 函数进去，只不过从来不返回。

那能不能让其他普通程序也运行于裸机呢？因此我把 eggos 的 kernel 抽离出来，作为一个单独的库，其他应用程序只要写下 `import _ "github.com/icexin/eggos"`，即可像 eggos 一样运行在裸机上。

即使是一个普通的 Go 程序，要在裸机上运行，编译过程还是有很多复杂的参数，因此我把这些编译过程封装了一下，编写了一个叫 `egg` 的程序，用这个程序来编译，打包，运行你自己编写的 Go 语言内核。

# 使用Go语言编写自己的unikernel

eggos 的主仓库里面带了几个示范的例子，在 [examples](https://github.com/icexin/eggos/tree/main/app/examples) 目录下，下面一一介绍这些例子。

其中需要注意的是，机器的环境准备如下：

- Go1.16.x
- Qemu模拟器

`egg`程序可以从 eggos 的 [Release](https://github.com/icexin/eggos/releases) 页面下载，或者运行 `go install github.com/icexin/eggos/cmd/egg@latest` 得到。

我们的例子都是在 qemu 模拟器里面运行，如果想在真实的机器上运行这些例子，可以参考动画那个例子，使用 iso 镜像在真机上运行。


## Hello Eggos

代码在 [hello world](https://github.com/icexin/eggos/tree/main/app/examples/helloworld/main.go)

第一个例子比较简单，就是简单输出 `hello eggos`，用来检查你的环境是否准备完毕

``` sh
$ cd helloworld
$ egg run
```

## Concurrent prime sieve

代码在 [prime sieve](https://github.com/icexin/eggos/tree/main/app/examples/prime-sieve/main.go)

第二个例子是 Go 官网经典的并发筛选素数的算法，这个例子展示了 eggos 使用 goroutine 的能力。

``` sh
$ cd prime-sieve
$ egg run
```

## HTTP 服务器

代码在 [http server](https://github.com/icexin/eggos/tree/main/app/examples/httpd/main.go)

第三个例子展示的是 eggos 的网络栈，可以运行 Go 标准库里面的 `http` 包

``` sh
$ cd httpd
$ egg run -p 8000:8000
```

打开浏览器，输入 http://127.0.0.1:8000/hello 即可访问。

因为我们是在 qemu 里面运行，因此做了端口映射，如果是裸机上运行，把 `127.0.0.1`替换成机器 IP 即可。

## 交互式程序

代码在 [repl](https://github.com/icexin/eggos/tree/main/app/examples/repl/main.go)

第四个例子展示了 eggos 的使用输入输出编写交互程序的例子，这个例子通过获取用户输入的字符串，计算其 `sha1`值，并输出到屏幕。

``` sh
$ cd repl
$ egg run
```

通过关闭 qemu 的窗口即可关闭程序。

## 图形与动画

代码在 [graphic](https://github.com/icexin/eggos/tree/main/app/examples/graphic/main.go)

第五个例子展示了 eggos 操作图像的能力。Go 的标准库里面的 `image` 有基础的图形处理能力，eggos 借助 `frame buffer`，具备基础的处理图像的能力。

这个例子的代码是从 《Go程序设计语言》里面借鉴来的，本来的例子是生成一个 [利萨茹曲线 (Lissajous curve)](https://zh.wikipedia.org/zh-hans/%E5%88%A9%E8%90%A8%E8%8C%B9%E6%9B%B2%E7%BA%BF)
的 GIF 动图，这里进行少稍微的改造，形成屏幕上的动画。

这个例子的命令比其他的多一些，之前的例子不牵扯到在屏幕上画图，我们使用qemu的 `-kernel` 参数加载内核即可，在这种模式下，qemu 是不会帮我们找到 `frame buffer`的。但在这个例子里面我们需要借助 grub 帮我们找到 `frame buffer`，因此生成了包含 grub 引导程序的 iso 镜像。

``` sh
$ cd graphic
$ egg pack -o graphic.iso
$ egg run graphic.iso
```

## Hack 系统调用

代码在 [syscall](https://github.com/icexin/eggos/tree/main/app/examples/syscall/main.go)

eggos 本身只包含了一些基础Linux系统调用，来让 Go 的 runtime 能正常运行，但是一些第三方库如果使用了 eggos 没有提供的系统调用可能不可以正常运行，因此 eggos 提供了自助注册系统调用的能力。 这个例子展示了如何自助注册系统调用。

``` sh
$ cd syscall
$ egg run
```

# 将内核运行在真机上

在 `图形与动画` 这个例子里面我们展示了如何将内核打包成一个 iso 文件，这个 iso 文件包含了引导程序，因此可以被真机识别并加载运行。

我们让裸机运行 iso 文件的通常的做法是把 iso 文件烧录到 U 盘或者移动硬盘里面，之后使用 U 盘或者移动硬盘插到电脑上，选择启动项运行。但在这里我推荐使用 [ventoy](https://www.ventoy.net/)来做启动盘，ventoy 只会格式化 U 盘一次，之后只用拷贝 iso 文件即可，非常方便。这是我在真机上的截图。

![bare-metal](https://i.imgur.com/YDlowOQ.gif)

# 结语

借助于 Go 语言的包管理功能，我们成功提供了一个 `libos` 的 Go 语言实现，让普通 Go 程序也可以跑在裸机上，这是一个非常有趣的尝试，我后面希望硬件驱动之类的也可以模块化，大家在 eggos 这个 unikernel 框架下定制自己的内核像 import 普通 Go package 一样简单！感谢阅读！
