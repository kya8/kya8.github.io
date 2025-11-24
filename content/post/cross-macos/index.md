---
title: 交叉编译到macOS
description: ...其实非常简单!
date: 2025-10-26
# lastmod: 
draft: false
tags:
    - macos
    - compilers
    - cross-platform
---

对于CI/CD系统, 或者本地的开发/测试而言, 为每个编译的Target系统准备一台机器或虚拟机显然有些繁琐. 最好, 我们能在一台机器上进行对多平台的交叉编译.
不过, 对于macOS目标, 目前网上的已有经验并不多.

# osxcross

大部分人搜索"cross compiling to macos"应该就会找到[osxcross](https://github.com/tpoechtrager/osxcross).

__众所周知, 交叉编译需要__:
1. 为target平台生成代码的编译器和工具链
2. target平台的一套SDK, 即头文件和一些基础库.

`osxcross`这个项目默认干了什么事呢?
* 编译`cctools-port`, 这是一个把Apple的`ar`, `ld,`, `otool`, `lipo`等工具链binary移植到了其他Unices的项目.
* 帮你获取并解包macOS的SDK. 其中, 会帮你编译`xar`, `pbzx`等若干工具.
* 至于最重要的编译器部分, 默认只是生成了一些wrapper, 调用你host系统上的Clang. 众所周知, Clang自己就自带交叉编译功能, 因此不需要额外编译一个不同target的cc.
    * 你也可以选择从头编译Clang或GCC

很简单吧! 其实弄懂原理后, 可以不需要`osxcross`

PS: `osxcross`仍然有一些小bug, 比如编译出的`ld`还依赖一个内部库, 需要你`LD_LIBRARY_PATH`一下, 比如一些wrapper脚本里变量设置的有问题.

# 自己动手

## 获取macOS SDK

先去Apple Developer官网直接下载Command Line Tools, 得到一个dmg文件, 如`Command_Line_Tools_for_Xcode_26.2.dmg`.

然后, 需要准备几个工具软件:
* 7z (p7zip或者新版的7zip)
* xar (一般发行版官方都有打包)
* [pbzx](https://github.com/NiklasRosenstein/pbzx.git), 另见我提的[修复Linux上编译的PR](https://github.com/NiklasRosenstein/pbzx/pull/9)

接下来解压:
```bash
7z e -so Command_Line_Tools_for_Xcode_26.2.dmg "Command Line Developer Tools/Command Line Tools*.pkg" > "Command Line Tools.pkg"

mkdir pkg_data out

xar -xf "Command Line Tools.pkg" -C pkg_data

for PKG in pkg_data/*.pkg; do
    pbzx -n "${PKG}/Payload" | (cd out && cpio -i)
done
```
解压完后, 就能在`out/Library/Developer/CommandLineTools/SDKs/`下面看到SDK的根目录了, 如`MacOSX15.4.sdk`. (下载的dmg版本不同, 具体位置可能不同). 其中含有头文件和基础库等.

## 配置Clang
这里就十分简单了. 只要你主机上的Clang/LLVM功能完整, 能够输出macOS的target.
```bash
clang --target=arm64-apple-darwin24.4 --sysroot=/path/to/SDKs/MacOSX15.4.sdk -fuse-ld=lld test.c -o test
```
其实就是设置一下target和sysroot. 另外linker需要用lld, 因为我们host上的GNU ld并不支持其它target.

### Universal binary
如果要编译Universal binary, 加入`-arch arm64 -arch x86_64`即可.
当然, 也可以分别编译后, 用`lipo`合并.

## 其它工具链软件
我们还需要Apple的其它工具链软件, 也就是`cctools-port`部分. 事实上, 我们用LLVM的工具链基本能完全替代:
* `otool` -> `llvm-otool`
* `ar` -> `llvm-ar`
* `strip` -> `llvm-strip`
* `lipo` -> `llvm-lipo`

LLVM工具链和Clang前端一样, 都原生支持交叉编译, 支持各平台的二进制格式.

## CMake
上面的配置足够编译一些简单的程序了. 对于CMake, 设置一些CMake的标准变量即可, 例如
```
-D CMAKE_CXX_COMPILER=clang
-D CMAKE_CXX_COMPILER_TARGET=arm64-apple-darwin24.4
-D CMAKE_LINKER_TYPE=LLD
-D CMAKE_SYSROOT=/path/to/SDKs/MacOSX15.4.sdk
-D CMAKE_OSX_DEPLOYMENT_TARGET=15.0 # 最低兼容的macOS版本
```
此外, 还有交叉编译应当设置的标准变量, 如:
```
-D CMAKE_SYSTEM_NAME=Darwin
-D CMAKE_SYSTEM_PROCESSOR=arm64
```

当然, 也可以把它们放进一个CMake toolchain文件.

## Windows主机?
Windows主机上用法也是一样简单, 获取SDK内容后, 去[LLVM github](https://github.com/llvm/llvm-project/releases/latest)直接下载安装Windows安装包. 记得把安装目录(bin)加到PATH里.

然后你就有了Windows上完整可用的Clang前端和LLVM工具链, 支持所有target平台. (其实这就是在Windows上使用Clang编译程序的最简单方法: 去官网直接下载安装LLVM安装包! 不是什么clang-cl或者MinGW Clang)

安装好的Clang默认是编译Windows原生程序, 并链接到MS的C/C++运行库.
与上面相同, 我们只要指定`--target`和`--sysroot`就能编译macOS程序了.

# 总结

由于工程中产生了交叉编译到macOS的需求, 花了半天时间研究, 但最终的方法其实非常简单明晰. 实际上Clang/LLVM承担了主要的heavy-lifting, 我们只需要获取到目标平台的SDK即可.


# Bonus: 编译到Windows
有了上述的经验, 很自然地, 我们会想到用类似的方法编译到Windows原生程序.
一直以来, 交叉编译到Windows最成熟, 也最常见的方案就是MinGW. 不过, MinGW的做法是自己提供了一套Windows API的SDK, 并且不能使用MSVC的C++ STL.

实际上, 交叉编译到Windows也只需要Clang/LLVM 加上 Windows的SDK. 后者需要我们自己去下载(需要同意MS的协议).
不过, 已经有人做了这个工作: [xwin](https://github.com/Jake-Shadle/xwin) [cargo-xwin](https://github.com/rust-cross/cargo-xwin)

这个项目是用于编译Rust到${arch}-pc-windows-msvc这些target, 实际上就是帮你自动下载Windows SDK, 并且修复一些目录/大小写问题, 然后你就可以直接使用了.
用法就是:
```sh
clang++ -target x86_64-pc-windows-msvc -fuse-ld=lld-link -I xwin/splat/crt/include -I xwin/splat/sdk/include/ucrt -I xwin/splat/sdk/include/um -I xwin/splat/sdk/include/shared -L xwin/splat/crt/lib/x86_64 -L xwin/splat/sdk/lib/um/x86_64 -L xwin/splat/sdk/lib/ucrt/x86_64 hello.cpp -o hello.exe
```
(或者也可以用clang-cl)
