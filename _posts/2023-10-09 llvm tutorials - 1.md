---
title: "llvm tutorials - 1"
categories:
  - blog
tags:
  - llvm
---


# llvm tutorials - $1$ -Getting Started with the LLVM System

-------

## 1. 从源码构建

1. 查看llvm

```bash
git clone https://github.com/llvm/llvm-project.git
# 为了减少clone时间，可进行浅克隆
# git clone --depth 1 https://github.com/llvm/llvm-project.git
```

2. 配置和构建llvm和clang

```bash
cd llvm-project
cmake -S llvm -B build -G Ninja -DCMAKE_BUILD_TYPE=Debug
ninja -C build check-llvm
# 仅构建而没有其他子项目
```

有可能会遇到`c++: fatal error: Killed signal terminated program cc1plus`的问题，请指定并行任务数量：

```bash
ninja -C build check-llvm -j 10
```

**一些常见的编译选项**

- `-DLLVM_ENABLE_PROJECTS='...'`— 您想要额外构建的 LLVM 子项目的分号分隔列表。可以包括以下任意项：clang、clang-tools-extra、lldb、lld、polly 或跨项目测试。

  例如，要构建 LLVM、Clang 和 LLD，请使用 `-DLLVM_ENABLE_PROJECTS="clang;lld"`.

- `-DCMAKE_INSTALL_PREFIX=directory`*— 为目录*指定要安装 LLVM 工具和库的完整路径名（默认`/usr/local`）。

- `-DCMAKE_BUILD_TYPE=type`— 控制构建的优化级别和调试信息。*类型*的有效选项有`Debug`、 `Release`、`RelWithDebInfo`和`MinSizeRel`。有关更多详细信息，请参阅[CMAKE_BUILD_TYPE](https://llvm.org/docs/CMake.html#cmake-build-type)。

- `-DLLVM_ENABLE_ASSERTIONS=ON`— 在启用断言检查的情况下进行编译（对于调试构建，默认为打开，对于所有其他构建类型，默认为关闭）。

- `-DLLVM_USE_LINKER=lld`— 使用[lld 链接器](https://lld.llvm.org/)进行链接，假设您的系统上安装了它。如果默认链接器很慢，这可以显着加快链接时间。

- `-DLLVM_PARALLEL_{COMPILE,LINK}_JOBS=N`— 限制同时并行运行的编译/链接作业的数量。这对于链接尤其重要，因为链接可能会使用大量内存。如果您在构建 LLVM 时遇到内存问题，请尝试设置此选项以限制同时运行的编译/链接作业的最大数量。

## 2. 补充

**如何获取最新的工具链？**

```bash
gcc_version=7.4.0
wget https://ftp.gnu.org/gnu/gcc/gcc-${gcc_version}/gcc-${gcc_version}.tar.bz2
wget https://ftp.gnu.org/gnu/gcc/gcc-${gcc_version}/gcc-${gcc_version}.tar.bz2.sig
wget https://ftp.gnu.org/gnu/gnu-keyring.gpg
signature_invalid=`gpg --verify --no-default-keyring --keyring ./gnu-keyring.gpg gcc-${gcc_version}.tar.bz2.sig`
if [ $signature_invalid ]; then echo "Invalid signature" ; exit 1 ; fi
tar -xvjf gcc-${gcc_version}.tar.bz2
cd gcc-${gcc_version}
./contrib/download_prerequisites
cd ..
mkdir gcc-${gcc_version}-build
cd gcc-${gcc_version}-build
$PWD/../gcc-${gcc_version}/configure --prefix=$HOME/toolchains --enable-languages=c,c++
make -j$(nproc)
make install
```

## 3. 目录布局

### 3.1 `llvm/cmake`

生成系统构建文件。

- `llvm/cmake/modules`

  构建 llvm 用户定义选项的配置。检查编译器版本和链接器标志。

- `llvm/cmake/platforms`

  Android NDK、iOS 系统和非 Windows 主机针对 MSVC 的工具链配置。

### 3.2 `llvm/examples`

- 一些简单的示例展示了如何使用 LLVM 作为自定义语言的编译器 - 包括降低、优化和代码生成。
- 万花筒教程：万花筒语言教程介绍了一个用于非平凡语言的漂亮的小型编译器的实现，包括手写的词法分析器、解析器、AST，以及使用 LLVM 的代码生成支持 - 静态（提前）和各种即时 (JIT) 编译方法。 [适合初学者的万花筒教程](https://llvm.org/docs/tutorial/MyFirstLanguageFrontend/index.html)。
- [BuildingAJIT： BuildingAJIT 教程](https://llvm.org/docs/tutorial/BuildingAJIT1.html)示例，展示 LLVM 的 ORC JIT API 如何与 LLVM 的其他部分交互。它还教授如何重新组合它们以构建适合您的用例的自定义 JIT。

### 3.3 `llvm/lib`


大多数源文件都在这里。通过将代码放入库中，LLVM 可以轻松地在工具之间共享代码。

### 3.4 `llvm/bindings`

包含 LLVM 编译器基础架构的绑定，以允许使用 C 或 C++ 以外的语言编写的程序利用 LLVM 基础架构。LLVM 项目提供 OCaml 和 Python 的语言绑定。

### 3.5 `llvm/projects`

项目严格来说不是 LLVM 的一部分，但随 LLVM 一起提供。这也是用于创建您自己的、利用 LLVM 构建系统的基于 LLVM 的项目的目录。

### 3.6 `llvm/tests`

对 LLVM 基础设施的功能和回归测试以及其他健全性检查。这些旨在快速运行并覆盖大量区域而不是详尽无遗。

### 3.7 `test-suite`

LLVM 的全面正确性、性能和基准测试套件。它位于 中，因为它包含各种许可证下的大量第三方代码。详细信息请参见[测试指南](https://llvm.org/docs/TestingGuide.html)文档。

### 3.8 `llvm/tools`

由上述库构建的可执行文件构成了用户界面的主要部分。您始终可以通过键入 来获取有关工具的帮助。以下是最重要的工具的简要介绍。更多详细信息请参见[命令指南](https://llvm.org/docs/CommandGuide/index.html)。

### 3.9 `llvm/utils`

用于处理 LLVM 源代码的实用程序；有些是构建过程的一部分，因为它们是基础设施部分的代码生成器。



