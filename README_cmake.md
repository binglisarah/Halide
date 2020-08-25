# Halide and CMake

This is a comprehensive guide to the three main usage stories of the Halide
CMake build.

1. Compiling or packaging Halide from source.
2. Building Halide programs using the official CMake package.
3. Contributing to Halide and updating the build files.

The following sections cover each in detail.

## Table of Contents

- [Getting started](#getting-started)
  - [Installing CMake](#installing-cmake)
  - [Installing dependencies](#installing-dependencies)
- [Building Halide with CMake](#building-halide-with-cmake)
  - [Basic build](#basic-build)
  - [Build options](#build-options)
- [Using Halide from your CMake build](#using-halide-from-your-cmake-build)
  - [A basic CMake project](#a-basic-cmake-project)
  - [JIT mode](#jit-mode)
  - [AOT mode](#aot-mode)
    - [Autoschedulers](#autoschedulers)
    - [RunGenMain](#rungenmain)
  - [Halide package documentation](#halide-package-documentation)
    - [Components](#components)
    - [Variables](#variables)
    - [Imported targets](#imported-targets)
    - [Functions](#functions)
      - [`add_halide_library`](#add_halide_library)
- [Contributing CMake code to Halide](#contributing-cmake-code-to-halide)
  - [General guidelines and best practices](#general-guidelines-and-best-practices)
    - [Prohibited commands list](#prohibited-commands-list)
    - [Prohibited variables list](#prohibited-variables-list)
  - [Adding tests](#adding-tests)
  - [Adding apps](#adding-apps)

# Getting started

This section covers installing a recent version of CMake and the correct
dependencies for building and using Halide.

## Installing CMake

Getting a recent version of CMake couldn't be easier, and there are multiple
good options on any system to do so. Generally one should always have the most
recent version of CMake installed system-wide. CMake is committed to backwards
compatibility and even the most recent release can build projects going back
over a decade.

### Cross-platform

The Python package manager `pip` packages the newest version of CMake at all
times. This might be the most convenient method since Python 3 is an optional
dependency for Halide, anyway.

```
$ pip install --upgrade cmake
```

See the [PyPI website](https://pypi.org/project/cmake) for more details.

### Windows

On Windows, there are three primary methods for installing an up-to-date CMake:

1. If you have **Visual Studio 2019** installed, you can get CMake 3.16 through
   the Visual Studio installer. This is the recommended way of getting CMake if
   you are able to use Visual Studio 2019. See the
   [Visual C++ documentation](https://docs.microsoft.com/en-us/cpp/build/cmake-projects-in-visual-studio?view=vs-2019)
   for more details.
2. If you use [Chocolatey](https://chocolatey.org/packages/cmake), its CMake
   package is kept up to date. It should be as simple as `choco install cmake`.
3. Otherwise you should install CMake directly from
   [Kitware's website](https://cmake.org/download/).

### macOS

On macOS, the [Homebrew](https://formulae.brew.sh/cask/cmake#default) cmake
package is kept up to date. Simply run

```
$ brew update
$ brew install cmake
```

to install the newest version of CMake. If your environment prevents you from
installing Homebrew, the binary release on
[Kitware's website](https://cmake.org/download/) is also a viable option.

### Ubuntu Linux

There are a few good ways to install a modern CMake on Ubuntu:

1. If you're on **Ubuntu Linux 20.04** (focal), then simply running
   `sudo apt install cmake` is sufficient.
2. If you are on an older Ubuntu release or would like to use a newer CMake,
   first try installing via the snap store: `snap install cmake`. Be sure you do
   not already have `cmake` installed via APT. The snap package automatically
   stays up to date.
3. For older versions of **Debian, Ubuntu, Mint, and derivatives**, Kitware
   provides an [APT repository](https://apt.kitware.com/) with up-to-date
   releases. Note that this is still useful for Ubuntu 20.04 because it will
   remain up to date.

For other Linux distributions, check with your distribution's package manager or
use pip as detailed above. Snap packages are also available.

On WSL 1, the snap service is not available; in this case, prefer to use the APT
repository. On WSL 2, all methods are available.

## Installing dependencies

We generally recommend using a package manager to fetch Halide's dependencies.
Except where noted, we recommend using vcpkg on Windows, Homebrew on macOS, and
apt on Ubuntu 20.04 LTS.

Only LLVM and Clang are _absolutely_ required to build Halide. Halide always
supports three LLVM versions: the current major version, the previous major
version, and trunk. For most users, we recommend using a binary release of LLVM
rather than building it yourself.

However, to run all of the tests and apps, an extended set is needed. This
includes lld, Python 3, libpng, libjpeg, Doxygen, OpenBLAS, ATLAS, and Eigen3.
While not required to build any part of Halide, we find that Ninja is the best
backend build tool across all platforms.

Note that CMake has many special variables for overriding the locations of
packages and executables. Consult the CMake documentation for these. Normally,
you should prefer to make sure your environment is sound. For instance, if you
want CMake to use a particular version of Python, create a virtual environment
and activate it _before_ configuring Halide.

### Windows

We assume you have vcpkg installed at `D:\vcpkg`. Follow the instructions in the
vcpkg README to install. First install LLVM via vcpkg.

```
D:\vcpkg> .\vcpkg install llvm[target-all,enable-assertions,clang-tools-extra]:x64-windows
D:\vcpkg> .\vcpkg install llvm[target-all,enable-assertions,clang-tools-extra]:x86-windows
```

This will also install Clang and LLD. The `enable-assertions` option is not
strictly necessary but will make debugging during development much smoother.
These builds will take a long time and a lot of disk space. After they are
built, it is safe to delete the intermediate build files and caches in
`%VCPKG_ROOT%\buildtrees` and `%APPDATA%\local\vcpkg`.

Then install the other libraries:

```
D:\vcpkg> .\vcpkg install libpng:x64-windows libjpeg-turbo:x64-windows openblas:x64-windows eigen3:x64-windows
D:\vcpkg> .\vcpkg install libpng:x86-windows libjpeg-turbo:x86-windows openblas:x86-windows eigen3:x86-windows
```

To build the documentation, you will need to install Doxygen. This can be done
either through Chocolatey or from their website.

```
> choco install doxygen
```

To build the Python bindings, you will need to install Python 3. This should be
done by running the official installer from the Python website. Be sure to
install system-wide (in `C:\Program Files`) and download the debugging symbols
through the installer. This will require using the "Advanced Installation"
workflow.

Once Python is installed, you can install the Python module dependencies either
globally or in a virtual environment by running

```
$ pip3 install -r .\python_bindings\requirements.txt
```

from the root of the repository.

If you would like to use Ninja, note that it is installed alongside CMake when
using the Visual Studio 2019 installer. Alternatively, you can install via
Chocolatey or place the pre-built binary from their website in the PATH.

```
> choco install ninja
```

### macOS

On macOS, it is possible to install all dependencies via Homebrew:

```
$ brew install llvm libpng libjpeg python@3.8 openblas doxygen ninja
```

The `llvm` package includes `clang`, `clang-format`, and `lld`, too.

### Ubuntu

Finally, on Ubuntu 20.04 LTS, you should install the following packages:

```
dev@ubuntu:~$ sudo apt install \
                  clang-tools lld llvm-dev libclang-dev liblld-10-dev \
                  libpng-dev libjpeg-dev libgl-dev \
                  python3-dev python3-numpy python3-scipy python3-imageio python3-pybind11 \
                  libopenblas-dev libeigen3-dev libatlas-base-dev \
                  doxygen ninja-build
```

# Building Halide with CMake

## Basic build

These instructions assume that your working directory is the Halide repo root.

### Windows

If you plan to use the Ninja generator, be sure to be in the developer command
prompt corresponding to your intended environment. Note that whatever your
intended target system (x86, x64, or arm), you must use the 64-bit _host tools_
because the 32-bit tools run out of memory during the linking step with LLVM.

You should either open the correct Developer Command Prompt directly or run the
`vcvarsall.bat` script with the correct argument, ie. one of the following:

```
D:\> "C:\Program Files (x86)\Microsoft Visual Studio\2019\Community\VC\Auxiliary\Build\vcvarsall.bat" x64
D:\> "C:\Program Files (x86)\Microsoft Visual Studio\2019\Community\VC\Auxiliary\Build\vcvarsall.bat" x64_x86
D:\> "C:\Program Files (x86)\Microsoft Visual Studio\2019\Community\VC\Auxiliary\Build\vcvarsall.bat" x64_arm
```

Then, assuming that vcpkg is installed to `D:\vcpkg`, simply run:

```
> cmake -G Ninja -DCMAKE_BUILD_TYPE=Release -DCMAKE_TOOLCHAIN_FILE=D:\vcpkg\scripts\buildsystems\vcpkg.cmake -S . -B build
> cmake --build .\build
```

Otherwise, if you wish to create a Visual Studio based build system, you can
configure with:

```
> cmake -G "Visual Studio 16 2019" -Thost=x64 -A x64 ^
        -DCMAKE_TOOLCHAIN_FILE=D:\vcpkg\scripts\buildsystems\vcpkg.cmake ^
        -S . -B build
> cmake --build .\build --config Release -j %NUMBER_OF_PROCESSORS%
```

or, for 32-bit:

```
> cmake -G "Visual Studio 16 2019" -Thost=x64 -A Win32 ^
        -DCMAKE_TOOLCHAIN_FILE=D:\vcpkg\scripts\buildsystems\vcpkg.cmake ^
        -S . -B build
> cmake --build .\build --config Release -j %NUMBER_OF_PROCESSORS%
```

In both cases, the `-Thost=x64` flag ensures that the correct host tools are
used.

### macOS and Linux

The instructions here are straightforward. Assuming your environment is set up
correctly, just run:

```
dev@host:~/Halide$ cmake -G Ninja -DCMAKE_BUILD_TYPE=Release -S . -B build
dev@host:~/Halide$ cmake --build ./build
```

## Installing

Once built, Halide will need to be installed somewhere before using it in a
separate project. On any platform, this means running the `cmake --install`
command in one of two ways. For a single-configuration generator (like Ninja),
run either:

```
dev@host:~/Halide$ cmake --install ./build --prefix /path/to/Halide-install
> cmake --install .\build --prefix X:\path\to\Halide-install
```

For a multi-configuration generator (like Visual Studio) run:

```
dev@host:~/Halide$ cmake --install ./build --prefix /path/to/Halide-install --config Release
> cmake --install .\build --prefix X:\path\to\Halide-install --config Release
```

Of course, make sure that you build the corresponding config before attempting
to install it.

## Build options

Halide reads and understands several options that can configure the build. The
following are the most consequential and control how Halide is actually
compiled.

| Option                       | Default               | Description                                                                                                      |
| ---------------------------- | --------------------- | ---------------------------------------------------------------------------------------------------------------- |
| `Halide_BUNDLE_LLVM`         | `OFF`                 | When building Halide as a static library, unpack the LLVM static libraries and add those objects to libHalide.a. |
| `Halide_SHARED_LLVM`         | `OFF`                 | Link to the shared version of LLVM. Not available on Windows.                                                    |
| `Halide_ENABLE_RTTI`         | _inherited from LLVM_ | Enable RTTI when building Halide. Recommended to be set to `ON`                                                  |
| `Halide_ENABLE_EXCEPTIONS`   | `ON`                  | Enable exceptions when building Halide                                                                           |
| `Halide_USE_CODEMODEL_LARGE` | `OFF`                 | Use the Large LLVM codemodel                                                                                     |
| `Halide_TARGET`              | _empty_               | The default target triple to use for `add_halide_library` (and the generator tests, by extension)                |

The following options are only available when building Halide directly, ie. not
through the `add_subdirectory` or `FetchContent` mechanisms. They control
whether non-essential targets (like tests and documentation) are built.

| Option                 | Default              | Description                                                                              |
| ---------------------- | -------------------- | ---------------------------------------------------------------------------------------- |
| `WITH_TESTS`           | `ON`                 | Enable building unit and integration tests                                               |
| `WITH_APPS`            | `ON`                 | Enable testing sample applications (run `ctest -L apps` to actually build and test them) |
| `WITH_PYTHON_BINDINGS` | `ON` if Python found | Enable building Python 3.x bindings                                                      |
| `WITH_DOCS`            | `OFF`                | Enable building the documentation via Doxygen                                            |
| `WITH_UTILS`           | `ON`                 | Enable building various utilities including the trace visualizer                         |
| `WITH_TUTORIALS`       | `ON`                 | Enable building the tutorials                                                            |

The following options control whether to build certain test subsets. They only
apply when `WITH_TESTS=ON`:

| Option                    | Default | Description                       |
| ------------------------- | ------- | --------------------------------- |
| `WITH_TEST_AUTO_SCHEDULE` | `ON`    | enable the auto-scheduling tests  |
| `WITH_TEST_CORRECTNESS`   | `ON`    | enable the correctness tests      |
| `WITH_TEST_ERROR`         | `ON`    | enable the expected-error tests   |
| `WITH_TEST_WARNING`       | `ON`    | enable the expected-warning tests |
| `WITH_TEST_PERFORMANCE`   | `ON`    | enable performance testing        |
| `WITH_TEST_OPENGL`        | `OFF`   | enable the OpenGL tests           |
| `WITH_TEST_GENERATOR`     | `ON`    | enable the AOT generator tests    |

The following options enable/disable various LLVM backends (they correspond to
LLVM component names):

| Option               | Default           | Description                                              |
| -------------------- | ----------------- | -------------------------------------------------------- |
| `TARGET_AARCH64`     | `ON` if available | Enable the AArch64 backend                               |
| `TARGET_AMDGPU`      | `ON` if available | Enable the AMD GPU backend                               |
| `TARGET_ARM`         | `ON` if available | Enable the ARM backend                                   |
| `TARGET_HEXAGON`     | `ON` if available | Enable the Hexagon backend                               |
| `TARGET_MIPS`        | `ON` if available | Enable the MIPS backend                                  |
| `TARGET_NVPTX`       | `ON` if available | Enable the NVidia PTX backend                            |
| `TARGET_POWERPC`     | `ON` if available | Enable the PowerPC backend                               |
| `TARGET_RISCV`       | `ON` if available | Enable the RISC V backend                                |
| `TARGET_WEBASSEMBLY` | `ON` if available | Enable the WebAssembly backend. Only valid for LLVM 11+. |
| `TARGET_X86`         | `ON` if available | Enable the x86 (and x86_64) backend                      |

The following options enable/disable various Halide-specific backends:

| Option                | Default | Description                            |
| --------------------- | ------- | -------------------------------------- |
| `TARGET_OPENCL`       | `ON`    | Enable the OpenCL-C backend            |
| `TARGET_OPENGL`       | `ON`    | Enable the OpenGL/GLSL backend         |
| `TARGET_METAL`        | `ON`    | Enable the Metal backend               |
| `TARGET_D3D12COMPUTE` | `ON`    | Enable the Direct3D 12 Compute backend |

The following options are WebAssembly-specific. They only apply when
`TARGET_WEBASSEMBLY=ON`:

| Option            | Default | Description                                                |
| ----------------- | ------- | ---------------------------------------------------------- |
| `WITH_WABT`       | `ON`    | Include WABT Interpreter for WASM testing                  |
| `WITH_WASM_SHELL` | `ON`    | Download a wasm shell (e.g. d8) for testing AOT wasm code. |

# Using Halide from your CMake build

This section assumes some basic familiarity with CMake but tries to be explicit
in all its examples. To learn more about CMake, consult the documentation and
engage with the community on the CMake Discourse.

Note: previous releases bundled a `halide.cmake` module that was meant to be
`include()`-ed into your project. This has been removed. Please upgrade to the
new package config module.

## A basic CMake project

There are two main ways to use Halide in your application: as a **JIT compiler**
for dynamic pipelines or an **ahead-of-time (AOT) compiler** for static
pipelines. CMake provides robust support for both use cases.

No matter how you intend to use Halide, you will need some basic CMake
boilerplate.

```
cmake_minimum_required(VERSION 3.16)
project(HalideExample)

set(CMAKE_CXX_STANDARD 11)  # or newer
set(CMAKE_CXX_STANDARD_REQUIRED YES)
set(CMAKE_CXX_EXTENSIONS NO)

find_package(Halide REQUIRED)
```

If Halide is not globally installed, you will need to add the root of the Halide
installation directory to `CMAKE_MODULE_PATH` at the CMake command line.

```
dev@ubuntu:~/myproj$ cmake -G Ninja -DCMAKE_BUILD_TYPE=Release -DCMAKE_MODULE_PATH="/path/to/Halide-install" -S . -B build
```

## JIT mode

To use Halide in JIT mode (like the
[tutorials](https://halide-lang.org/tutorials/tutorial_introduction.html) do,
for example), you can simply link to `Halide::Halide`.

```
# ... same project setup as before ...
add_executable(my_halide_app main.cpp)
target_link_libraries(my_halide_app PRIVATE Halide::Halide)
```

Then `Halide.h` will be available to your code and everything should just work.
That's it!

## AOT mode

Using Halide in AOT mode is more complicated so we'll walk through it step by
step. Note that this only applies to Halide generators, so it might be useful to
re-read the
[tutorial](https://halide-lang.org/tutorials/tutorial_lesson_15_generators.html)
on generators. Assume (like in the tutorial) that you have a source file named
`my_generators.cpp` and that in it you have generator classes `MyFirstGenerator`
and `MySecondGenerator` with registered names `my_first_generator` and
`my_second_generator` respectively.

Then the first step is to add a **generator executable** to your build:

```
# ... same project setup as before ...
add_executable(my_generators my_generators.cpp)
target_link_libraries(my_generators PRIVATE Halide::Generator)
```

Using the generator executable, we can add a Halide library corresponding to
`MyFirstGenerator`.

```
# ... continuing from above
add_halide_library(my_first_generator FROM my_generators)
```

This will create a
[STATIC](https://cmake.org/cmake/help/latest/command/add_library.html) library
target in CMake that corresponds to the output of running your generator. The
second generator in the file requires generator parameters to be passed to it.
These are also easy to handle:

```
# ... continuing from above
add_halide_library(my_second_generator FROM my_generators
                   PARAMS parallel=false scale=3.0 rotation=ccw output.type=uint16)
```

Adding multiple configurations is easy, too:

```
# ... continuing from above
add_halide_library(my_second_generator_2 FROM my_generators
                   GENERATOR my_second_generator
                   PARAMS scale=9.0 rotation=ccw output.type=float32)

add_halide_library(my_second_generator_3 FROM my_generators
                   GENERATOR my_second_generator
                   PARAMS parallel=false output.type=float64)
```

Here, we had to specify which generator to use (`my_second_generator`) since it
uses the target name by default. The functions in these libraries will be named
after the target names, `my_second_generator_2` and `my_second_generator_3`, by
default, but it is possible to control this via the `FUNCTION_NAME` parameter.

Each one of these targets, `<GEN>`, carries an associated `<GEN>.runtime`
target, which is also a STATIC library containing the Halide runtime. It is
transitively linked through `<GEN>` to targets that link to `<GEN>`. On an
operating system like Linux, where weak linking is available, this is not an
issue. However, on Windows, this will fail due to symbol redefinitions. In these
cases, you must declare that two Halide libraries share a runtime, like so:

```
# ... updating above
add_halide_library(my_second_generator_2 FROM my_generators
                   GENERATOR my_second_generator
                   USE_RUNTIME my_first_generator.runtime
                   PARAMS scale=9.0 rotation=ccw output.type=float32)

add_halide_library(my_second_generator_3 FROM my_generators
                   GENERATOR my_second_generator
                   USE_RUNTIME my_first_generator.runtime
                   PARAMS parallel=false output.type=float64)
```

This will even work correctly when different combinations of targets are
specified for each halide library. A "greatest common denominator" target will
be chosen that is compatible with all of them.

### Autoschedulers

When the autoschedulers are included in the release package, they are very
simple to apply to your own generators. For example, we could update the
definition of the `my_first_generator` library above to use the `Adams2019`
autoscheduler:

```
add_halide_library(my_second_generator FROM my_generators
                   AUTOSCHEDULER Halide::Adams2019
                   PARAMS auto_schedule=true)
```

### RunGenMain

Halide provides a generic driver for generators to be used during development
for benchmarking and debugging. Suppose you have a generator executable called
`my_gen` and a generator within called `my_filter`. Then you can pass a variable
name to the `REGISTRATION` parameter of `add_halide_library` which will contain
the name of a generated C++ source that should be linked to `Halide::RunGenMain`
and `my_filter`.

For example:

```
add_halide_library(my_filter FROM my_gen
                   REGISTRATION filter_reg_cpp)
add_executable(runner ${filter_reg_cpp})
target_link_libraries(runner PRIVATE my_filter Halide::RunGenMain)
```

Then you can run, debug, and benchmark your generator through the `runner`
executable.

## Halide package documentation

### Components

The Halide package script understands a handful of optional components when
loading the package.

First, if you plan to use the Halide Image IO library, you will want to include
the `png` and `jpeg` components when loading Halide.

Second, Halide releases can contain a variety of configurations: static, shared,
debug, release, etc. CMake handles Debug/Release configurations automatically,
but generally only allows one type of library to be loaded.

The package understands two components, `static` and `shared`, that specify
which type of library you would like to load. For example, if you want to make
sure that you link against shared Halide, you can write:

```
find_package(Halide REQUIRED COMPONENTS shared)
```

If the shared libraries are not available, this will result in a failure.

If no component is specified, then the `Halide_SHARED_LIBS` variable is checked.
If it is defined and set to true, then the shared libraries will be loaded or
the package loading will fail. Similarly, if it is defined and set to false, the
static libraries will be loaded.

If no component is specified and `Halide_SHARED_LIBS` is _not_ defined, then the
`BUILD_SHARED_LIBS` variable will be inspected. If it is **not defined** or
**defined and set to true**, then it will attempt to load the shared libs and
fall back to the static libs if they are not available. Similarly, if
`BUILD_SHARED_LIBS` is **defined and set to false**, then it will try the static
libs first then fall back to the shared libs.

### Variables

Variables that control package loading:

- `Halide_SHARED_LIBS` -- override `BUILD_SHARED_LIBS` when loading the Halide
  package via `find_package`. Has no effect when using Halide via
  `add_subdirectory` as a Git or FetchContent submodule.

Variables set by the package:

- `Halide_VERSION` -- The full version string of the loaded Halide package
- `Halide_VERSION_MAJOR` -- The major version of the loaded Halide package
- `Halide_VERSION_MINOR` -- The minor version of the loaded Halide package
- `Halide_VERSION_PATCH` -- The patch version of the loaded Halide package
- `Halide_VERSION_TWEAK` -- The tweak version of the loaded Halide package
- `Halide_HOST_TARGET` -- The Halide target triple corresponding to "host" for
  this build.
- `Halide_ENABLE_EXCEPTIONS` -- Whether Halide was compiled with exception
  support
- `Halide_ENABLE_RTTI` -- Whether Halide was compiled with RTTI

### Imported targets

Halide defines the following targets that are available to users:

- `Halide::Halide` -- this is the JIT-mode library to use when using Halide from
  C++.
- `Halide::Generator` -- this is the target to use when defining a generator
  executable. It supplies a `main()` function.
- `Halide::Runtime` -- adds include paths to the Halide runtime headers
- `Halide::Tools` -- adds include paths to the Halide tools, including the
  benchmarking utility.
- `Halide::ImageIO` -- adds include paths to the Halide image IO utility and
  sets up dependencies to PNG / JPEG if they are available.
- `Halide::RunGenMain` -- used with the `REGISTRATION` parameter of
  `add_halide_library` to create simple runners and benchmarking tools for
  Halide libraries.

The following targets are not guaranteed to be available:

- `Halide::Python` -- this is a Python 3 module that can be referenced as
  `$<TARGET_FILE:Halide::Python>` when setting up Python tests or the like from
  CMake.
- `Halide::Adams19` -- the Adams et.al. 2019 autoscheduler (no GPU support)
- `Halide::Li18` -- the Li et.al. 2018 gradient autoscheduler (limited GPU
  support)

### Functions

Currently, only one function is defined:

#### `add_halide_library`

This is the main function for managing generators in AOT compilation. The full
signature follows:

```
add_halide_library(<target> FROM <generator-target>
                   [GENERATOR generator-name]
                   [FUNCTION_NAME function-name]
                   [USE_RUNTIME hl-target]
                   [PARAMS param1 [param2 ...]]
                   [TARGETS target1 [target2 ...]]
                   [FEATURES feature1 [feature2 ...]]
                   [PLUGINS plugin1 [plugin2 ...]]
                   [AUTOSCHEDULER scheduler-name]
                   [GRADIENT_DESCENT]
                   [C_BACKEND]
                   [REGISTRATION OUTVAR]
                   [<extra-output> OUTVAR])

extra-output = ASSEMBLY | BITCODE | COMPILER_LOG | CPP_STUB
             | FEATURIZATION | LLVM_ASSEMBLY | PYTHON_EXTENSION
             | PYTORCH_WRAPPER | SCHEDULE | STMT | STMT_HTML
```

This function creates a called `<target>` corresponding to running the
`<generator-target>` (an executable target which links to `Halide::Generator`)
one time, using command line arguments derived from the other parameters.

The arguments `GENERATOR` and `FUNCTION_NAME` default to `<target>`. They
correspond to the `-g` and `-f` command line flags, respectively.

If `USE_RUNTIME` is not specified, this function will create another target
called `<target>.runtime` which corresponds to running the generator with `-r`
and a compatible list of targets. This runtime target is an INTERFACE dependency
of `<target>`. If multiple runtime targets need to be linked together, setting
`USE_RUNTIME` to another Halide library, `<target2>` will prevent the generation
of `<target>.runtime` and instead use `<target2>.runtime`.

Parameters can be passed to a generator via the `PARAMS` argument. Parameters
should be space-separated. Similarly, `TARGETS` is a space-separated list of
targets for which to generate code in a single function. They must all share the
same platform/bits/os triple (eg. `arm-32-linux`). Features that are in common
among all targets, including device libraries (like `cuda`) should go in
`FEATURES`.

Every element of `TARGETS` must begin with the same `arch-bits-os` triple. This
function understands two _meta-triples_, `host` and `cmake`. The meta-triple
`host` is equal to the `arch-bits-os` triple used to compile Halide. On
platforms that support running both 32 and 64 bit programs, this will not
necessarily equal the true host platform. Additionally, `host` will include all
supported instruction set extensions. The meta-triple `cmake` is equal to the
`arch-bits-os` of the current CMake target and is suitable for cross-compiling.
If this triple matches `host`, then `host` will be used instead. When `TARGETS`
is empty, `cmake` is the default.

To set the default autoscheduler, set the `AUTOSCHEDULER` argument to a target
named like `Namespace::Scheduler`, for example `Halide::Adams19`. This will set
the `-s` flag on the generator command line to `Scheduler` and add the target to
the list of plugins. Additional plugins can be loaded by setting the `PLUGINS`
argument. If the argument to `AUTOSCHEDULER` does not contain `::` or it does
not name a target, it will be passed to the `-s` flag verbatim.

If `GRADIENT_DESCENT` is set, then the module will be built suitably for
gradient descent calculation in TensorFlow or PyTorch. See
`Generator::build_gradient_module()` for more documentation. This corresponds to
passing `-d 1` at the generator command line.

If the `C_BACKEND` option is set, this command will produce a `STATIC` library
target (not `IMPORTED`). It does this by supplying `c_source` to the `-e` flag
in place of `static_library` and then calling the currently configured C++
compiler. Note that a `<target>.runtime` target is _not_ created in this case,
and the `USE_RUNTIME` option is ignored. Other options work as expected.

If `REGISTRATION` is set, the path to the generated `.registration.cpp` file
will be set in `OUTVAR`. This can be used to generate a runner for a Halide
library that is useful for benchmarking and testing, as documented above. This
is equivalent to setting `-e registration` at the generator command line.

Lastly, each of the `extra-output` arguments directly correspond to an extra
output (via `-e`) from the generator. The value `OUTVAR` names a variable into
which a path (relative to `CMAKE_CURRENT_BINARY_DIR`) to the extra file will be
written.

## Cross compiling

Cross-compiling is a fraught topic in CMake when some binaries must be built for
the host platform. Unfortunately, this is almost always the case with Halide
generator executables. Each project will be set up differently and have
different requirements, but here are some suggestions for effective use of CMake
in these scenarios.

### Use a super-build

A CMake super-build consists of breaking down a project into sub-projects that
are isolated by toolchain. The basic structure is to have an outermost project
that only coordinates the sub-builds via the
[ExternalProject](https://cmake.org/cmake/help/latest/module/ExternalProject.html)
module.

One would then use Halide to build a generator executable in one self-contained
project, then export that target to be used in a separate project. The second
project would be configured with the target toolchain and would call
`add_halide_library` with no `TARGETS` option and set `FROM` equal to the name
of the imported generator executable. Obviously, this is a significant increase
in complexity over a typical CMake project.

### Use ExternalProject directly

A lighter weight alternative to the above is to use `ExternalProject` directly
in your parent build. Configure the parent build with the target toolchain, and
configure the inner project to use the host toolchain. Then, manually create an
IMPORTED target for your generator executable and call `add_halide_library` as
described above.

The main drawback of this approach is that creating accurate IMPORTED targets is
difficult since predicting the names and locations of your binaries across all
possible platform and CMake project generators is difficult. In particular, it
is hard to predict executable extensions in cross-OS builds.

### Use an emulator or run on device

The
[`CMAKE_CROSSCOMPILING_EMULATOR`](https://cmake.org/cmake/help/latest/variable/CMAKE_CROSSCOMPILING_EMULATOR.html)
variable allows one to specify a command _prefix_ to run a target-system binary
on the host machine. One could set this to a custom shell script that uploads
the generator executable, runs it on the device and copies back the results.

### Bypass CMake

The previous two options ensure that the targets generated by
`add_halide_library` will be _normal_ static libraries. This approach does not
use ExternalProject, but instead produces `IMPORTED` targets. The main drawback
of `IMPORTED` targets is that they are considered second-class in CMake. In
particular, they cannot be installed with the typical `install(TARGETS)`
command. Instead, they must be installed using `install(FILES)` and the
`$<TARGET_FILE:>` generator expression.

# Contributing CMake code to Halide

When contributing new CMake code to Halide, keep in mind that the minimum
version is 3.16. Therefore, it is possible (and indeed required) to use modern
CMake best practices.

Like any large and complex system with a dedication to preserving backwards
compatibility, CMake is difficult to learn and full of traps. While not
comprehensive, the following serves as a guide for writing quality CMake code
and outlines the code quality expectations we have as they apply to CMake.

## General guidelines and best practices

The following are some common mistakes that lead to subtly broken builds.

- **Reading the build directory.** While setting up the build, the build
  directory should be considered _write only_. Using the build directory as a
  read/write temporary directory is acceptable as long as all temp files are
  cleaned up by the end of configuration.
- **Prefer generator expressions.** Declarative is better than imperative and
  this is no exception. Conditionally adding to a target property can leak
  unwanted details about the build environment into packages. Some information
  is not accurate or available except via generator expressions, eg. the build
  configuration.
- **Using the wrong variable.** `CMAKE_SOURCE_DIR` doesn't always point to the
  Halide source root. When someone uses Halide via FetchContent, it will point
  to _their_ source root instead. The correct variable is `Halide_SOURCE_DIR`.
  If you want to know if the compiler is MSVC, check it directly with the `MSVC`
  variable; don't use `WIN32`. That will be wrong when compiling with clang on
  Windows.
- **Using directory properties.** Directory properties have vexing behavior and
  are essentially deprecated from CMake 3.0+. Propagating target properties is
  the way of the future.
- **Using the wrong visibility.** Target properties can be `PRIVATE`,
  `INTERFACE`, or both (aka `PUBLIC`). Pick the most conservative one for each
  scenario.

### Prohibited commands list

As mentioned above, using directory properties is brittle and they are therefore
_not allowed_. The following functions may not appear in any new CMake code.

| Command                             | Alternative                                               |
| ----------------------------------- | --------------------------------------------------------- |
| `add_compile_definitions`           | Use `target_compile_definitions`                          |
| `add_compile_options`               | Use `target_compile_options`                              |
| `add_definitions`                   | Use `target_compile_definitions`                          |
| `add_link_options`                  | Use `target_link_options`, but prefer not to use either   |
| `get_directory_property`            | Use cache variables or target properties                  |
| `get_property(... DIRECTORY)`       | Use cache variables or target properties                  |
| `include_directories`               | Use `target_include_directories`                          |
| `link_directories`                  | Use `target_link_libraries`                               |
| `link_libraries`                    | Use `target_link_libraries`                               |
| `remove_definitions`                | Generator expressions with `target_compile_definitions`   |
| `set_directory_properties`          | Use cache variables or target properties                  |
| `set_property(... DIRECTORY)`       | Use cache variables or target properties                  |
| `target_link_libraries(target lib)` | Use `target_link_libraries` _with a visibility specifier_ |

If common properties need to be grouped together, use an INTERFACE target
(better) or write a function (worse). There are also several functions that are
disallowed for other reasons:

| Command                         | Reason                                                      | Alternative                                                   |
| ------------------------------- | ----------------------------------------------------------- | ------------------------------------------------------------- |
| `aux_source_directory`          | Interacts poorly with incremental builds and Git            | List source files explicitly                                  |
| `build_command`                 | CTest internal function                                     | Use CTest build-and-test mode via `CTEST_COMMAND`             |
| `cmake_host_system_information` | Usually misleading information.                             | Inspect toolchain variables and use generator expressions.    |
| `cmake_policy(... OLD)`         | OLD policies are deprecated by definition.                  | Instead, fix the code to work with the new policy.            |
| `create_test_sourcelist`        | We use our own unit testing solution                        | See the [adding tests](#adding-tests) section.                |
| `define_property`               | Adds unnecessary complexity                                 | Use a cache variable. Exceptions under special circumstances. |
| `enable_language`               | Halide is C/C++ only                                        | `FindCUDAToolkit` or `FindCUDA`, appropriately guarded.       |
| `file(GLOB ...)`                | Interacts poorly with incremental builds and Git            | List source files explicitly                                  |
| `fltk_wrap_ui`                  | Halide does not use FLTK                                    | None                                                          |
| `include_external_msproject`    | Halide must remain portable                                 | Write a CMake package config file or find module.             |
| `include_guard`                 | Use of recursive inclusion is not allowed                   | Write (recursive) functions.                                  |
| `include_regular_expression`    | Changes default dependency checking behavior                | None                                                          |
| `load_cache`                    | Superseded by FetchContent/ExternalProject                  | Use aforementioned modules                                    |
| `macro`                         | CMake macros are not hygienic and are therefore error-prone | Use functions instead.                                        |
| `site_name`                     | Privacy: do not want leak host name information             | Provide a cache variable, generate a unique name.             |
| `variable_watch`                | Debugging helper                                            | None. Not needed in production.                               |

Lastly, do not introduce any dependencies via `find_package` without broader
approval. Confine dependencies to the `dependencies/` subtree.

### Prohibited variables list

Any variables that are specific to languages that are not enabled should, of
course, be avoided. But of greater concern are variables that are easy to misuse
or should not be overridden for our end-users. The following (non-exhaustive)
list of variables shall not be used in code merged into master.

| Variable                        | Reason                                        | Alternative                                                                       |
| ------------------------------- | --------------------------------------------- | --------------------------------------------------------------------------------- |
| `CMAKE_ROOT`                    | Code smell                                    | Rely on `find_package` search options; include `HINTS` if necessary               |
| `CMAKE_DEBUG_TARGET_PROPERTIES` | Debugging helper                              | None                                                                              |
| `CMAKE_FIND_DEBUG_MODE`         | Debugging helper                              | None                                                                              |
| `CMAKE_RULE_MESSAGES`           | Debugging helper                              | None                                                                              |
| `CMAKE_VERBOSE_MAKEFILE`        | Debugging helper                              | None                                                                              |
| `CMAKE_BACKWARDS_COMPATIBILITY` | Deprecated                                    | None                                                                              |
| `CMAKE_BUILD_TOOL`              | Deprecated                                    | `${CMAKE_COMMAND} --build` or `CMAKE_MAKE_PROGRAM`                                |
| `CMAKE_CACHEFILE_DIR`           | Deprecated                                    | `CMAKE_BINARY_DIR`, but see below                                                 |
| `CMAKE_CFG_INTDIR`              | Deprecated                                    | `$<CONFIG>`, `$<TARGET_FILE:..>`, target resolution of `add_custom_command`, etc. |
| `CMAKE_CL_64`                   | Deprecated                                    | `CMAKE_SIZEOF_VOID_P`                                                             |
| `CMAKE_COMPILER_IS_*`           | Deprecated                                    | `CMAKE_<LANG>_COMPILER_ID`                                                        |
| `CMAKE_HOME_DIRECTORY`          | Deprecated                                    | `CMAKE_SOURCE_DIR`, but see below                                                 |
| `CMAKE_DIRECTORY_LABELS`        | Directory property                            | None                                                                              |
| `CMAKE_BUILD_TYPE`              | Only applies to single-config generators.     | `$<CONFIG>`                                                                       |
| `CMAKE_*_FLAGS*` (w/o `_INIT`)  | User-only                                     | Write a toolchain file with the corresponding `_INIT` variable                    |
| `CMAKE_COLOR_MAKEFILE`          | User-only                                     | None                                                                              |
| `CMAKE_ERROR_DEPRECATED`        | User-only                                     | None                                                                              |
| `CMAKE_CONFIGURATION_TYPES`     | We only support the four standard build types | None                                                                              |

Of course feel free to insert debugging helpers _while developing_ but please
remove them before review. Finally, the following variables are allowed, but
their use must be motivated:

| Variable               | Reason                                              | Alternative                                                            |
| ---------------------- | --------------------------------------------------- | ---------------------------------------------------------------------- |
| `CMAKE_SOURCE_DIR`     | Points to global source root, not Halide's.         | `Halide_SOURCE_DIR` or `PROJECT_SOURCE_DIR`                            |
| `CMAKE_BINARY_DIR`     | Points to global build root, not Halide's           | `Halide_BINARY_DIR` or `PROJECT_BINARY_DIR`                            |
| `CMAKE_MAKE_PROGRAM`   | CMake abstracts over differences in the build tool. | Prefer CTest's build and test mode or CMake's `--build` mode           |
| `CMAKE_CROSSCOMPILING` | Often misleading.                                   | Inspect relevant variables directly, eg. `CMAKE_SYSTEM_NAME`           |
| `BUILD_SHARED_LIBS`    | Could override user setting                         | None, but be careful to restore value when overriding for a dependency |

Any use of these functions and variables will block a PR.

## Adding tests

When adding a file to any of the folders under `test`, be aware that CI expects
that every `.c` and `.cpp` appears in the `CMakeLists.txt` file _on its own
line_, possibly as a comment. This is to avoid globbing and also to ensure that
added files are not missed.

For most test types, it should be as simple as adding to the existing lists,
which must remain in alphabetical order. Generator tests are trickier, but
following the existing examples is a safe way to go.

## Adding apps

If you're contributing a new app to Halide: great! Thank you! There are a few
guidelines you should follow when writing a new app.

- Write the app as if it were a top-level project. You should call
  `find_package(Halide)` and set the C++ version to 14.
- Call `enable_testing()` and add a small test that runs the app.
- Don't assume you have access to a GPU. Write your schedules to be robust to
  varying buildbot hardware.
- If you rely on any additional packages, don't include them as `REQUIRED`,
  instead test to see if their targets are available and, if not, call
  `return()` before creating any targets. In this case, print a
  `message(STATUS "[SKIP] ...")`, too.
- Look at the existing apps for examples.
- Test your app with ctest before opening a PR. Apps are built as part of the
  test, rather than the main build.
