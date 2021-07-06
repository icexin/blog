---
title: "Dlv支持GDB Stub"
date: 2021-07-06T02:08:29+08:00
draft: false
---

# 前言

开发 [eggos](https://github.com/icexin/eggos) 的过程中一直使用 GDB 做调试，诸多功能都非常不方便，比如对 goroutine 的打印，变量的展开等等。dlv 是 go 专属的调试器，也是 go 官方推荐的：

> Note that Delve is a better alternative to GDB when debugging Go programs built with the standard toolchain. It understands the Go runtime, data structures, and expressions better than GDB

遗憾的是，dlv 提供的调试方式都不能用到 eggos 的调试上。dlv 只支持两种方式来调试，一种是`Launch`，即是 dlv 启动 go 进程，另外一种是`Attach`到一个进程 pid 上。qemu 提供了一个 GDB stub 服务，这个服务遵从 [GDB Serial Protocol](https://sourceware.org/gdb/onlinedocs/gdb/Remote-Protocol.html)，dlv 虽然也支持远程调试，但用的是自己的私有协议。因此长期以来我调试 eggos 一直是打印日志加 `runtime-gdb.py` 脚本的方式。

实际上，dlv 是支持 GDB 远程协议的。最近我浏览 dlv 代码的时候发现它有 [gdbserver.go](https://github.com/go-delve/delve/blob/v1.6.1/pkg/proc/gdbserial/gdbserver.go)这样的文件，阅读代码后了解到，原来 dlv 是为了支持在 mac 上运行 lldb 支持了 `GDB Serial Protocol`。但是在 dlv 的后端里面没有对远程连接 GDB stub 的选项。

# 编译使用

既然代码在，只是缺少包装，我开了一个 `GDB` 分支，加入了对于 GDB 后端的支持。按照如下方式编译

``` sh
$ git clone --branch gdb https://github.com/icexin/delve.git
$ cd delve
$ go install ./cmd/dlv
```

放在`$HOME/go/bin`下的 dlv 二进制即可以直连实现 `GDB Serial Protocol` 的服务了。

使用方式如下：

首先启动 qemu
``` sh
$ qemu-system-i386 -s -S -kernel multiboot.elf
```

使用 dlv attach上去

``` sh
$ DLV_GOOS=linux DLV_GOARCH=386 dlv attach --backend gdbstub 1234 ./kernel.elf
```
之后就可以按照 dlv 正常 debug go 的方式进行了。

其中参数解释如下：

- `--beckend`指定 `gdbstub`，这个是我新加的 backend
- `1234` 为 GDB 协议监听的端口
- `DLV_GOOS`和`DLV_GOARCH`分别指定运行二进制的操作系统平台和 CPU 架构，跟 `GOOS` 和 `GOARCH` 类似，如果没有这两个环境变量，默认是 `linux`和`386`。

# vscode集成

一个 vscode 集成的例子，具体见 eggos 项目的 `.vscode/launch.json`。

``` json
"configurations": [
        {
            "name": "dlv-gdb",
            "type": "go",
            "request": "attach",
            "mode": "local",
            "cwd": "${workspaceFolder}",
            "debugAdapter": "legacy",
            "processId": 1234,
            "backend": "gdbstub",
            "dlvFlags": [
                "${workspaceRoot}/kernel.elf",
            ],
        },
]
```