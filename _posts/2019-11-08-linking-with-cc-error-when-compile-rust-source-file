---
layout: post
title: Rust 编译报错 linking with `cc` failed: exit code: 1 
category: Rust
---

编译 Rust 的 hello world 时执行命令 `rustc main.rs` 报错

```shell
$ rustc main.rs
error: linking with `cc` failed: exit code: 1
  |
  = note: "cc" "-m64" "-L" "/Users/xxxx/.rustup/toolchains/stable-x86_64-apple-darwin/lib/rustlib/x86_64-apple-darwin/lib" "main.main.7rcbfp3g-cgu.0.rcgu.o" "main.main.7rcbfp3g-cgu.1.rcgu.o" "main.main.7rcbfp3g-cgu.2.rcgu.o" "main.main.7rcbfp3g-cgu.3.rcgu.o" "main.main.7rcbfp3g-cgu.4.rcgu.o" "main.main.7rcbfp3g-cgu.5.rcgu.o" "-o" "main" "main.4s37gsrti678ik8u.rcgu.o" "-Wl,-dead_strip" "-nodefaultlibs" "-L" "/Users/xxxx/.rustup/toolchains/stable-x86_64-apple-darwin/lib/rustlib/x86_64-apple-darwin/lib" "/Users/xxxx/.rustup/toolchains/stable-x86_64-apple-darwin/lib/rustlib/x86_64-apple-darwin/lib/libstd-ec578e0d01ad5d6e.rlib" "/Users/xxxx/.rustup/toolchains/stable-x86_64-apple-darwin/lib/rustlib/x86_64-apple-darwin/lib/libpanic_unwind-5412e5af11009a97.rlib" "/Users/xxxx/.rustup/toolchains/stable-x86_64-apple-darwin/lib/rustlib/x86_64-apple-darwin/lib/libhashbrown-03db0718fbd4a443.rlib" "/Users/xxxx/.rustup/toolchains/stable-x86_64-apple-darwin/lib/rustlib/x86_64-apple-darwin/lib/librustc_std_workspace_alloc-8df90fdde44531fa.rlib" "/Users/xxxx/.rustup/toolchains/stable-x86_64-apple-darwin/lib/rustlib/x86_64-apple-darwin/lib/libbacktrace-080b75c76cf389d3.rlib" "/Users/xxxx/.rustup/toolchains/stable-x86_64-apple-darwin/lib/rustlib/x86_64-apple-darwin/lib/libbacktrace_sys-954947c96c071ed1.rlib" "/Users/xxxx/.rustup/toolchains/stable-x86_64-apple-darwin/lib/rustlib/x86_64-apple-darwin/lib/librustc_demangle-9a1775bac6aabe20.rlib" "/Users/xxxx/.rustup/toolchains/stable-x86_64-apple-darwin/lib/rustlib/x86_64-apple-darwin/lib/libunwind-71147793b4cdc412.rlib" "/Users/xxxx/.rustup/toolchains/stable-x86_64-apple-darwin/lib/rustlib/x86_64-apple-darwin/lib/libcfg_if-9fc81eecc6136c9a.rlib" "/Users/xxxx/.rustup/toolchains/stable-x86_64-apple-darwin/lib/rustlib/x86_64-apple-darwin/lib/liblibc-4b64712313317864.rlib" "/Users/xxxx/.rustup/toolchains/stable-x86_64-apple-darwin/lib/rustlib/x86_64-apple-darwin/lib/liballoc-1bcd644d1289b2fb.rlib" "/Users/xxxx/.rustup/toolchains/stable-x86_64-apple-darwin/lib/rustlib/x86_64-apple-darwin/lib/librustc_std_workspace_core-16c65b3b16ee989d.rlib" "/Users/xxxx/.rustup/toolchains/stable-x86_64-apple-darwin/lib/rustlib/x86_64-apple-darwin/lib/libcore-7dd67903be10326a.rlib" "/Users/xxxx/.rustup/toolchains/stable-x86_64-apple-darwin/lib/rustlib/x86_64-apple-darwin/lib/libcompiler_builtins-b5923fb6eca9603a.rlib" "-lSystem" "-lresolv" "-lc" "-lm"
  = note: xcrun: error: invalid active developer path (/Library/Developer/CommandLineTools), missing xcrun at: /Library/Developer/CommandLineTools/usr/bin/xcrun


error: aborting due to previous error

```

主要是错误是 `error: linking with `cc` failed: exit code: 1`。
操作系统 macOS Catalina，检查 gcc 版本

```shell
$ gcc --version
xcrun: error: invalid active developer path (/Library/Developer/CommandLineTools), missing xcrun at: /Library/Developer/CommandLineTools/usr/bin/xcrun
```

解决办法

```shell
xcode-select --install
```
安装完成以后，再检查 gcc 版本

```shell
$ gcc --version
Configured with: --prefix=/Library/Developer/CommandLineTools/usr --with-gxx-include-dir=/Library/Developer/CommandLineTools/SDKs/MacOSX.sdk/usr/include/c++/4.2.1
Apple clang version 11.0.0 (clang-1100.0.33.8)
Target: x86_64-apple-darwin19.0.0
Thread model: posix
InstalledDir: /Library/Developer/CommandLineTools/usr/bin
```

再次执行 `rustc main.rs` 成功
