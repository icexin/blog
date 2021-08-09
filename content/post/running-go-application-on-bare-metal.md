---
title: "Running Go Application on Bare Metal"
date: 2021-08-09T22:17:27+08:00
draft: false
---

In the process of writing [eggos](https://github.com/icexin/eggos), I found that removing the hardware initialization code, the main program is actually an ordinary Go program. Like other Go programs, the main function is used as the entry point, but it never returns.

Can other ordinary programs also run on bare metal? The answer is yes. I extracted the kernel code of eggos as a separate Go library. Other applications can run on bare metal like eggos by writing `import _ "github.com/icexin/eggos"`.

If an ordinary Go program wants to run on bare metal, in addition to import eggos, it also needs to pass many complex parameters to the compiler. I encapsulated these compilation processes and wrote a program called `egg`. In this way, you can use `egg` to compile, package, and run the Go language kernel written by yourself.

# Use Go to write your own unikernel

There are several examples in the main code repository of eggos, in the [examples](https://github.com/icexin/eggos/tree/main/app/examples) directory, the following will introduce these examples one by one.

It should be noted that all examples depend on the following:

- Go1.16.x
- The Qemu emulator

The `egg` program can be downloaded from eggos's [Release](https://github.com/icexin/eggos/releases) page, or run `go install github.com/icexin/eggos/cmd/egg@latest` to get it.

Our examples are all running in the qemu emulator. If you want to run these examples on a real machine, you can refer to the `graphic` example and use the iso image to run on the real machine.

## Hello Eggos

The code is in [hello world](https://github.com/icexin/eggos/tree/main/app/examples/helloworld/main.go)

The first example is relatively simple, it simply outputs `hello eggos` to check whether your environment is ready.

``` sh
$ cd helloworld
$ egg run
```

## Concurrent prime sieve

The code is in [prime sieve](https://github.com/icexin/eggos/tree/main/app/examples/prime-sieve/main.go)

The second example is the classic algorithm for `Concurrent prime sieve` on the Go official website. This example demonstrates the ability of eggos to use goroutines.

``` sh
$ cd prime-sieve
$ egg run
```

## HTTP server

The code is in [http server](https://github.com/icexin/eggos/tree/main/app/examples/httpd/main.go)

The third example shows the eggos network stack, which can run the `http` package in the Go standard library

``` sh
$ cd httpd
$ egg run -p 8000:8000
```

Open the browser and enter http://127.0.0.1:8000/hello to access.

Because we are running in qemu, port mapping is used. If running on bare metal, just replace `127.0.0.1` with the machine IP.

## Interactive program

The code is in [repl](https://github.com/icexin/eggos/tree/main/app/examples/repl/main.go)

The fourth example shows an example of using input and output to write an interactive program. This example obtains the string entered by the user, calculates its sha1 value, and outputs it to the screen.

``` sh
$ cd repl
$ egg run
```

You can close the program by closing the qemu window.

## Graphics and Animation

The code is in [graphic](https://github.com/icexin/eggos/tree/main/app/examples/graphic/main.go)

The fifth example demonstrates eggos' ability to manipulate images. The `image` package in Go's standard library has basic graphics processing capabilities, and eggos has basic image processing capabilities with the help of [Frame Buffer](https://en.wikipedia.org/wiki/Framebuffer).

The code of this example is borrowed from the "Go Programming Language". The original example is to generate a [Lissajous curve](https://en.wikipedia.org/wiki/Lissajous_curve)
GIF animation, here is a little modification to form an animation on the screen.

This example has more commands than others. The previous example does not involve drawing pictures on the screen. We can use qemu's `-kernel` parameter to load the kernel. In this mode, qemu will not help us find the `frame buffer`. But in this example, we need to use grub to help us find the `frame buffer`, so an iso image containing the grub boot program is generated.

``` sh
$ cd graphic
$ egg pack -o graphic.iso
$ egg run graphic.iso
```

## Hacking system call

The code is in [syscall](https://github.com/icexin/eggos/tree/main/app/examples/syscall/main.go)

Eggos itself only contains some basic Linux system calls to allow Go runtime to run normally, but some third-party libraries may not run normally if they use system calls that are not provided by eggos, so eggos provides the ability to register system calls. This example shows how to register system calls.

``` sh
$ cd syscall
$ egg run
```

# Run the kernel on the real machine

In the example of `Graphics and Animation`, we showed how to package the kernel into an ISO file. This ISO file contains the bootloader, so it can be recognized and loaded and run by the real machine.

The usual way we let the bare metal run the ISO file is to burn the ISO file into the USB drive, then insert the USB drive into the computer, and select the USB drive startup item to run. But here I recommend using [ventoy](https://www.ventoy.net/) to make the boot disk. Ventoy will only format the USB disk once, then only need to copy the ISO file to the USB drive, which is very convenient. This is a screenshot of my machine.

![bare-metal](https://i.imgur.com/YDlowOQ.gif)

# Conclusion

With the help of the Go package management, we have successfully provided a Go language implementation of `libos` so that ordinary Go programs can also run on bare metal. This is a very interesting attempt. I hope that the hardware driver can also be modularized into a Go package, so that you can customize your own kernel under the eggos framework as easy as importing a normal Go package! 

Happy hacking!