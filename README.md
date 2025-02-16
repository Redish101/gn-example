# GN Quick Start Guide

本仓库是文章[GN(Generate Ninja) 快速入门指南](https://blog.redish101.top/artical/gn-quick-start-guide)的最终成品源码仓库。

GN(Generate Ninja)是 Google 开发的元构建系统（类似 cmake），用以从配置文件生成`build.ninja`文件。

gn 在 Chromium、Fuchsia 以及 openharmony 等项目中均有使用，但是似乎远远比不上 cmake 使用的广泛，互联网上的资料也较少。这篇文章将通过使用 gn 构建一个简单的基于 glfw + glad 的 cpp 程序来介绍 gn 的使用。

## 0. 安装

### ninja

由于 gn 最终生成的是 ninja 的构建文件，所以需要安装 ninja。

linux 下可以通过[包管理器](https://github.com/ninja-build/ninja/wiki/Pre-built-Ninja-packages)

```bash
# Arch
pacman -S ninja

# Ubuntu/Debian
apt-get install ninja-build

# Fedora
dnf install ninja-build
```

或[可执行文件](https://github.com/ninja-build/ninja/releases)安装。

macOS 下可以通过[Homebrew](https://brew.sh/)：

```bash
brew install ninja
```

也可以通过[可执行文件](https://github.com/ninja-build/ninja/releases)安装。

windows 下可以通过使用winget：

```powershell
winget install Ninja-build.Ninja
```

或通过[可执行文件](https://github.com/ninja-build/ninja/releases)安装。

### gn

gn 可以通过包管理器安装，如在 ubuntu 下：

```bash
sudo apt install generate-ninja
```

在 arch 下：

```bash
sudo pacman -S gn
```

或通过编译安装：

```bash
git clone https://gn.googlesource.com/gn
cd gn
python build/gen.py # --allow-warning if you want to build with warnings.
ninja -C out
```

### 编辑器插件

如果想要有更顺畅的编写体验，可以安装gn的编辑器插件以使编译器支持gn的语法高亮、自动补全等功能。

在 vscode 中，可以安装插件[GN Language Server](https://marketplace.visualstudio.com/items?itemName=msedge-dev.gnls)。

## 1. 创建工程及配置编译工具链

首先在项目根目录下创建一个名为`.gn`的文件，用于声明工作区的根目录，并引入构建配置。

```
# .gn
buildconfig = "//build/BUILDCONFIG.gn"
```

> gn 中可以通过//描述路径，//表示工作区根目录

gn **并没有** 像 cmake 默认配置好了编译工具链，需要自己进行配置。可以参考gn源码仓库示例项目的工具链配置。

```
# build/toolchain/BUILD.gn

oolchain("gcc") {
  tool("cc") {
    depfile = "{{output}}.d"
    command = "gcc -MMD -MF $depfile {{defines}} {{include_dirs}} {{cflags}} {{cflags_c}} -c {{source}} -o {{output}}"
    depsformat = "gcc"
    description = "CC {{output}}"
    outputs =
        [ "{{source_out_dir}}/{{target_output_name}}.{{source_name_part}}.o" ]
  }
  tool("cxx") {
    depfile = "{{output}}.d"
    command = "g++ -MMD -MF $depfile {{defines}} {{include_dirs}} {{cflags}} {{cflags_cc}} -c {{source}} -o {{output}}"
    depsformat = "gcc"
    description = "CXX {{output}}"
    outputs =
        [ "{{source_out_dir}}/{{target_output_name}}.{{source_name_part}}.o" ]
  }
  tool("alink") {
    command = "ar rcs {{output}} {{inputs}}"
    description = "AR {{target_output_name}}{{output_extension}}"
    outputs =
        [ "{{target_out_dir}}/{{target_output_name}}{{output_extension}}" ]
    default_output_extension = ".a"
    output_prefix = "lib"
  }
  tool("solink") {
    soname = "{{target_output_name}}{{output_extension}}"  # e.g. "libfoo.so".
    sofile = "{{output_dir}}/$soname"
    rspfile = soname + ".rsp"
    if (is_mac) {
      os_specific_option = "-install_name @executable_path/$sofile"
      rspfile_content = "{{inputs}} {{solibs}} {{libs}}"
    } else {
      os_specific_option = "-Wl,-soname=$soname"
      rspfile_content = "-Wl,--whole-archive {{inputs}} {{solibs}} -Wl,--no-whole-archive {{libs}}"
    }
    command = "g++ -shared {{ldflags}} -o $sofile $os_specific_option @$rspfile"
    description = "SOLINK $soname"
    # Use this for {{output_extension}} expansions unless a target manually
    # overrides it (in which case {{output_extension}} will be what the target
    # specifies).
    default_output_extension = ".so"
    # Use this for {{output_dir}} expansions unless a target manually overrides
    # it (in which case {{output_dir}} will be what the target specifies).
    default_output_dir = "{{root_out_dir}}"
    outputs = [ sofile ]
    link_output = sofile
    depend_output = sofile
    output_prefix = "lib"
  }
  tool("link") {
    outfile = "{{target_output_name}}{{output_extension}}"
    rspfile = "$outfile.rsp"
    if (is_mac) {
      command = "g++ {{ldflags}} -o $outfile @$rspfile {{solibs}} {{libs}}"
    } else {
      command = "g++ {{ldflags}} -o $outfile -Wl,--start-group @$rspfile {{solibs}} -Wl,--end-group {{libs}}"
    }
    description = "LINK $outfile"
    default_output_dir = "{{root_out_dir}}"
    rspfile_content = "{{inputs}}"
    outputs = [ outfile ]
  }
  tool("stamp") {
    command = "touch {{output}}"
    description = "STAMP {{output}}"
  }
  tool("copy") {
    command = "cp -af {{source}} {{output}}"
    description = "COPY {{source}} {{output}}"
  }
}
```

如果需要使用clang：

```
# build/toolchain/BUILD.gn

toolchain("clang") {
  tool("cc") {
    depfile = "{{output}}.d"
    command = "clang -MMD -MF $depfile {{defines}} {{include_dirs}} {{cflags}} {{cflags_c}} -c {{source}} -o {{output}}"
    depsformat = "gcc"
    description = "CC {{output}}"
    outputs =
        [ "{{source_out_dir}}/{{target_output_name}}.{{source_name_part}}.o" ]
  }
  tool("cxx") {
    depfile = "{{output}}.d"
    command = "clang++ -MMD -MF $depfile {{defines}} {{include_dirs}} {{cflags}} {{cflags_cc}} -c {{source}} -o {{output}}"
    depsformat = "gcc"
    description = "CXX {{output}}"
    outputs =
        [ "{{source_out_dir}}/{{target_output_name}}.{{source_name_part}}.o" ]
  }
  tool("alink") {
    command = "ar rcs {{output}} {{inputs}}"
    description = "AR {{target_output_name}}{{output_extension}}"
    outputs =
        [ "{{target_out_dir}}/{{target_output_name}}{{output_extension}}" ]
    default_output_extension = ".a"
    output_prefix = "lib"
  }
  tool("solink") {
    soname = "{{target_output_name}}{{output_extension}}"  # e.g. "libfoo.so"
    sofile = "{{output_dir}}/$soname"
    rspfile = soname + ".rsp"
    if (is_mac) {
      os_specific_option = "-install_name @executable_path/$sofile"
      rspfile_content = "{{inputs}} {{solibs}} {{libs}}"
    } else {
      os_specific_option = "-Wl,-soname=$soname"
      rspfile_content = "-Wl,--whole-archive {{inputs}} {{solibs}} -Wl,--no-whole-archive {{libs}}"
    }
    command = "clang++ -shared {{ldflags}} -o $sofile $os_specific_option @$rspfile"
    description = "SOLINK $soname"
    default_output_extension = ".so"
    default_output_dir = "{{root_out_dir}}"
    outputs = [ sofile ]
    link_output = sofile
    depend_output = sofile
    output_prefix = "lib"
  }
  tool("link") {
    outfile = "{{target_output_name}}{{output_extension}}"
    rspfile = "$outfile.rsp"
    if (is_mac) {
      command = "clang++ {{ldflags}} -o $outfile @$rspfile {{solibs}} {{libs}}"
    } else {
      command = "clang++ {{ldflags}} -o $outfile -Wl,--start-group @$rspfile {{solibs}} -Wl,--end-group {{libs}}"
    }
    description = "LINK $outfile"
    default_output_dir = "{{root_out_dir}}"
    rspfile_content = "{{inputs}}"
    outputs = [ outfile ]
  }
  tool("stamp") {
    command = "touch {{output}}"
    description = "STAMP {{output}}"
  }
  tool("copy") {
    command = "cp -af {{source}} {{output}}"
    description = "COPY {{source}} {{output}}"
  }
}
```

然后在构建配置中指定工具链：

```
if (target_os == "") {
  target_os = host_os
}
if (target_cpu == "") {
  target_cpu = host_cpu
}
if (current_cpu == "") {
  current_cpu = target_cpu
}
if (current_os == "") {
  current_os = target_os
}
is_linux = host_os == "linux" && current_os == "linux" && target_os == "linux"
is_mac = host_os == "mac" && current_os == "mac" && target_os == "mac"
# All binary targets will get this list of configs by default.
_shared_binary_target_configs = [ "//build:compiler_defaults" ]
# Apply that default list to the binary target types.
set_defaults("executable") {
  configs = _shared_binary_target_configs
  # Executables get this additional configuration.
  configs += [ "//build:executable_ldconfig" ]
}
set_defaults("static_library") {
  configs = _shared_binary_target_configs
}
set_defaults("shared_library") {
  configs = _shared_binary_target_configs
}
set_defaults("source_set") {
  configs = _shared_binary_target_configs
}
set_default_toolchain("//build/toolchain:clang")
# 若使用gcc
set_default_toolchain("//build/toolchain:gcc")
```

上方的工具链配置需要在`build/BUILD.gn`中声明一些配置：

```
config("compiler_defaults") {
  if (current_os == "linux") {
    cflags = [
      "-fPIC",
      "-pthread",
    ]
  }
}
config("executable_ldconfig") {
  if (!is_mac) {
    ldflags = [
      "-Wl,-rpath=\$ORIGIN/",
      "-Wl,-rpath-link=",
    ]
  }
}
```

> 在gn中，`config`是一个用于定义配置的构建工具。它允许你定义一些变量，这些变量和常量可以在构建过程中被引用。

## 2. 声明构建配置

### 引入 glad

将下载的glad项目放入`thirdparty/glad`下，书写构建配置。

```
# thirdparty/glfw/BUILD.gn

config("glad_inner_headers") {
    include_dirs = [
        "include",
    ]
}

static_library("glad") {
    sources = [
        "src/glad.c",
    ]
    include_dirs = [
        "include",
    ]
    public = [
        "include/glad/glad.h",
        "include/KHR/khrplatform.h"
    ]
    public_configs = [
        ":glad_inner_headers",
    ]
}
```

构建配置中，`config`用于定义一些变量，这些变量可以在构建过程中被引用。`static_library`用于声明一个静态库，静态库是一个包含多个源文件的库，这些源文件被编译成静态链接库。

`static_library`中，`sources`用于声明静态库的源文件，`include_dirs`用于声明静态库的头文件目录，`public`用于声明静态库的头文件，`public_configs`中的字段会应用到所有依赖于此目标的目标。

### 引入 glfw

类似的，可以为 glfw 书写配置文件：

```
# thirdparty/glfw/BUILD.gn
# 修改自 ohos/third_party/glfw/BUILD.gn

# Copyright (c) 2021-2024 Huawei Device Co., Ltd.
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

config("glfw_inner_headers") {
  include_dirs = [ "include" ]
}

config("glfw_config_mac") {
  cflags = [ "-Wno-sign-compare" ]
}

static_library("glfw") {
  defines = [ "PREVIEW" ]
  include_dirs = [ "src" ]
  public = [
    "include/GLFW/glfw3.h",
    "include/GLFW/glfw3native.h",
  ]

  sources = [
    "src/context.c",
    "src/init.c",
    "src/input.c",
    "src/monitor.c",
    "src/null_init.c",
    "src/null_joystick.c",
    "src/null_joystick.h",
    "src/null_monitor.c",
    "src/null_platform.h",
    "src/null_window.c",
    "src/osmesa_context.c",
    "src/platform.c",
    "src/vulkan.c",
    "src/window.c",
  ]

  public_configs = [ ":glfw_inner_headers" ]

  if (current_os == "mingw") {
    sources += [
      "src/egl_context.c",
      "src/egl_context.h",
      "src/wgl_context.c",
      "src/wgl_context.h",
      "src/win32_init.c",
      "src/win32_joystick.c",
      "src/win32_joystick.h",
      "src/win32_module.c",
      "src/win32_monitor.c",
      "src/win32_platform.h",
      "src/win32_thread.c",
      "src/win32_thread.h",
      "src/win32_time.c",
      "src/win32_time.h",
      "src/win32_window.c",
    ]

    defines += [ "_GLFW_WIN32" ]

    libs = [
      "gdi32",
      "opengl32",
    ]
  }
  if (current_os == "mac") {
    sources += [
      "src/cocoa_init.m",
      "src/cocoa_joystick.h",
      "src/cocoa_joystick.m",
      "src/cocoa_monitor.m",
      "src/cocoa_platform.h",
      "src/cocoa_time.c",
      "src/cocoa_time.h",
      "src/cocoa_window.m",
      "src/egl_context.c",
      "src/egl_context.h",
      "src/nsgl_context.m",
      "src/posix_module.c",
      "src/posix_poll.c",
      "src/posix_poll.h",
      "src/posix_thread.c",
      "src/posix_time.c",
      "src/posix_time.h",
    ]
    include_dirs += [ "deps" ]
    if (defined(enable_gn_2021)) {
      frameworks = [
        "Cocoa.framework",
        "IOKit.framework",
        "CoreFoundation.framework",
        "CoreVideo.framework",
        "QuartzCore.framework",
      ]
    } else {
      libs = [
        "Cocoa.framework",
        "IOKit.framework",
        "CoreFoundation.framework",
        "CoreVideo.framework",
        "QuartzCore.framework",
      ]
    }

    cflags = [
      "-Wno-deprecated-declarations",
      "-Wno-objc-multiple-method-names",
      "-DNS_FORMAT_ARGUMENT(A)=",
    ]
    public_configs += [ ":glfw_config_mac" ]
    defines += [ "_GLFW_COCOA" ]
  }
  if (current_os == "linux") {
    sources += [
      "src/egl_context.c",
      "src/glx_context.c",
      "src/linux_joystick.c",
      "src/linux_joystick.h",
      "src/posix_module.c",
      "src/posix_poll.c",
      "src/posix_poll.h",
      "src/posix_thread.c",
      "src/posix_time.c",
      "src/posix_time.h",
      "src/x11_init.c",
      "src/x11_monitor.c",
      "src/x11_platform.h",
      "src/x11_window.c",
      "src/xkb_unicode.c",
      "src/xkb_unicode.h",
    ]
    cflags_c = [
      "-Wno-sign-compare",
      "-Wno-missing-field-initializers",
    ]
    libs = [
      "rt",
      "dl",
      "X11",
      "Xcursor",
      "Xinerama",
      "Xrandr",
    ]
    defines += [ "_GLFW_X11" ]
    defines += [ "USE_DUMMPY_XINPUT2" ]
  }
}
```

gn 中可以通过`if`实现针对不同目标平台的条件编译。

在 gn 中可以通过`declare_args`声明构建参数，如：

```
# build/args.gni

declare_args() {
    is_dev = false
}
```

在`BUILD.gn`中引入参数：

```
import("//build/args.gni")
```

在目标配置中根据不同参数设置不同的`defines`参数：

```
import("//build/args.gni")

executable("hello") {
    source = [ "src/hello.cpp" ]

    defines = []

    if (is_dev) {
        defines += [ "IS_DEV" ]
    }
}
```

可以使程序在不同配置下进行不同的操作：

```cpp
#include <iostream>

int main(int argc, char** argv)
{
    #ifdef IS_DEV
    std::cout << "dev mode" << std::endl;
    #endif

    return 0;
}
```

在生成构建文件时，可以通过`gn gen --args="is_dev=true"`指定参数。

### 可执行文件

在这篇文章中，我们使用一个简单的基于 glfw + glad 的图形应用程序来演示如何使用 gn 构建。

```
# app/BUILD.gn

executable("app") {
    sources = [
        "src/main.cpp",
    ]
    deps = [
        "//thirdparty/glad:glad",
        "//thirdparty/glfw:glfw"
    ]
}

```

通过`executable`定义可执行文件目标，通过`deps`指定依赖。

`//`指向项目的根目录，也就是`.gn`文件的位置，`:`后是目标名称。

完成上述配置后，可以在根目录`BUILD.gn`定义一个`group("all")`目标作为默认的编译目标：

```
group("all") {
    deps = [
        "//app:app"
    ]
}
```

## 3. 构建

构建过程分为两个步骤：

1. 使用gn生成ninja.build文件
2. 使用ninja编译生成可执行文件

通过`gn gen out`可以在`out`下生成`build.ninja`文件，通过`ninja -C out`编译生成可执行文件。也可以通过`ninja -C out -t <target>`指定编译目标。

对于本例子，执行`ninja -C out`后将会在`out`下生成可执行文件`app`。

执行`gn args out`会弹出一个文本编辑器，用以指定编译参数。也可以通过`gn gen out --args="<args>"`指定编译参数。通过`gn args --list`可以查看所有的编译参数。

## 总结

由上面的例子可以看出，gn相比于cmake，从零开始新建项目较为繁琐，而且并没有cmake的应用广泛，对于本文的例子如果引入glfw，需要重新编写gn文件，对于个人开发者以及小型项目来说可能并不方便，所以gn的应用场景更多是规模较大的项目中，更为清晰的描述构建。

本文的最终源码已经托管在[GitHub](https://github.com/Redish101/gn-example)