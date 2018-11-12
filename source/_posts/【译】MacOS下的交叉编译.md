---
title: 【译】MacOS下的交叉编译
abbrlink: 18577
date: 2018-09-08 20:09:52
banner: 'https://saekiraku-1253597019.coscd.myqcloud.com/image/z4c9z.png'
category: Linux
tags: Linux
---

原文地址：[Easy Windows and Linux cross-compilers for MacOS](https://blog.filippo.io/easy-windows-and-linux-cross-compilers-for-macos/)

<!-- more -->

---

> 本文章基于以下CC协议进行知识共享：署名（BY）-非商业性使用（NC）-禁止演绎（ND）

---

在 MacOS 下编译 Linux 和 Windows 的可执行程序，需要使用 Homebrew 安装 C/C++ 交叉编译工具链：

```
brew install FiloSottile/musl-cross/musl-cross
brew install mingw-w64
```

---

交叉编译 C 和 C++ 是非常痛苦的。

当你使用 Golang 时，你只需要设置一个环境变量即可（译注：CC），对于 C 你则需要一整套离散的工具链，这也许需要一些中间件来进行构建，并且你需要非常清楚你每一步的目的。

# musl-cross-make

幸运的是，`Rich Felker` 设计了一套 `Makefile` 来构建基于 [musl](http://www.musl-libc.org) 库的交叉编译器 —— [musl-cross-make](https://github.com/richfelker/musl-cross-make)。它使用了一些补丁，使其在 MacOS 上可以良好的运行。

musl-cross-make 完美实现了独立的交叉编译器，所以你无需关注正确的库路径，也不需要了解在何处存放工具链。同时，它可以编译到任意架构的目标 Linux 上。

需要注意的是，它是基于 musl 的 C 标准库。这意味着编译后的二进制库文件只能运行在基于 musl 的系统上，例如：Alpine。即便如此，如果你通过传入 `--static` 参数来构建静态二进制文件便可以运行在任何系统上，包括 `scratch` Docker 容器中。musl 是一个特殊的工程，它可以完整的支持静态编译二进制文件（译注：跨平台），所以并不推荐使用 `glibc` 来编译。

# homebrew-musl-cross

到目前为止，我非常喜欢 `Homebrew`。它允许你在一个完美的沙盒中构建应用，并且只有可执行文件链接到了环境变量中，类似 [GNU Stow](https://www.gnu.org/software/stow/) 的风格。同时，它也能管理资源并提供非常强大的开发工具。

所以，我将 `musl-cross-make` 封装成了一个 `Homebrew Formula` 。我花了很长时间来构建这个项目，但是最终它成为了一个完整的交叉编译工具链，并且以不同的前缀链接进了 `/usr/local/bin`。例如： `x86_64-linux-musl-gcc` 。

```
brew install FiloSottile/musl-cross/musl-cross  
```

它包含了一个预编译过的 `Homebrew Bottle` ，可以运行在 `High Sierra` 系统中。所以如果你想自己通过源码构建，使用 `brew install --build-from-source` 即可、

其他的架构也是被支持的。比如基于树莓派的交叉编译器：

```
brew install FiloSottile/musl-cross/musl-cross --without-x86_64 --with-arm-hf
```

你也可以使用 `--with-i486`（x86 32-bit），`--with-aarch64`（ARM 64-bit），`--with-arm`（ARM soft-float）和 `--with-mips` 等开关参数。

# 应用到 Golang 和 Rust

为了交叉编译基于 `cgo` 的项目，你可以设置 `CC` 和 `CCX` 环境变量标志来构建到 `x86_64-linux-musl-gcc` 和 `x86_64-linux-musl-g++`（或其他相关架构）。该环境变量标志应放在 `GOOS` 和 `GOARCH` 两个环境变量标志之前。

```
CC=x86_64-linux-musl-gcc GOOS=linux GOARCH=amd64 CGO_ENABLED=1 go build -a -v main.go
// 译注：自己额外增加的范例
```

如果使用这个工具链来交叉编译 Rust 项目，则添加以下设置到 `.cargo/config`：

```
[target.x86_64-unknown-linux-musl]
linker = "x86_64-linux-musl-gcc"
```

这里有一份更完整的 Rust 交叉编译教程：[Cross compile and link a static binary on macos for linux with cargo and rust](https://chr4.org/blog/2017/03/15/cross-compile-and-link-a-static-binary-on-macos-for-linux-with-cargo-and-rust/)

# mingw-w64

对于 Windows 系统下的交叉编译，有一个 `Mingw-w64` 的 Formula 在 homebrew-core 中，所以你可以使用 `brew install mingw-w64` 直接安装。

这个 GCC 工具链的前缀为 `x86_64-w64-mingw32-` 和 `i686-w64-mingw32-`。

如果你觉得交叉编译比你想象中的有趣，也许你会想[关注我的 Twitter](https://twitter.com/FiloSottile)

# 译后记

第一次在 Golang 项目中使用到了一个基于 CGO 的库 —— go-sqlite3，并且交叉编译失败，在查找解决方案时发现了一篇不错的文章，因此而翻译。这篇文章不错的地方一是包含了实际问题的解决方案；二是原文作者的讲解十分清晰，翻译起来也十分轻松；三是在原作者在解决问题的基础上，又能简单讲解交叉编译的一些基础知识，对于交叉编译后续的深入研究，可以作为不错的切入点。