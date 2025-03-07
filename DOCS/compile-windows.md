Compiling for Windows
=====================

Compiling for Windows is supported with MinGW-w64. This can be used to produce
both 32-bit and 64-bit executables, and it works for building on Windows and
cross-compiling from Linux and Cygwin. MinGW-w64 is available from:
https://www.mingw-w64.org/

While building a complete MinGW-w64 toolchain yourself is possible, there are a
few build environments and scripts to help ease the process, such as MSYS2 and
MXE. Note that MinGW environments included in Linux distributions are often
broken, outdated and useless, and usually don't use MinGW-w64.

**Warning**: the original MinGW (https://osdn.net/projects/mingw/) is unsupported.

Cross-compilation
=================

When cross-compiling, it is recommended to use a meson crossfile to setup
the cross compiling environment. A minimal example is included below:

```ini
[binaries]
c = 'x86_64-w64-mingw32-gcc'
cpp = 'x86_64-w64-mingw32-g++'
ar = 'x86_64-w64-mingw32-ar'
strip = 'x86_64-w64-mingw32-strip'
exe_wrapper = 'wine64'

[host_machine]
system = 'windows'
cpu_family = 'x86_64'
cpu = 'x86_64'
endian = 'little'
```

An alternative example for cross compiling for ARM64 platforms (on an MSYS2 ARM64 host) is also included:

```ini
[binaries]
c = 'clang'
cpp = 'clang'
ld = 'llvm-ld'
ar = 'llvm-lib'
strip = 'llvm-strip'
cmake = 'cmake'
pkgconfig = 'pkgconf'

[host_machine]
system = 'windows'
cpu_family = 'aarch64'
cpu = 'aarch64'
endian = 'little'
```

See [meson's documentation](https://mesonbuild.com/Cross-compilation.html) for
more information.

[MXE](https://mxe.cc) makes it very easy to bootstrap a complete MingGW-w64
environment from a Linux machine. See a working example below.

Alternatively, you can try [mpv-winbuild-cmake](https://github.com/shinchiro/mpv-winbuild-cmake),
which bootstraps a MinGW-w64 environment and builds mpv and dependencies.

Example with MXE
----------------

```bash
# Before starting, make sure you install MXE prerequisites. MXE will download
# and build all target dependencies, but no host dependencies. For example,
# you need a working compiler, or MXE can't build the crosscompiler.
#
# Refer to
#
#    https://mxe.cc/#requirements
#
# Scroll down for disto/OS-specific instructions to install them.

# Download MXE. Note that compiling the required packages requires about 1.4 GB
# or more!

cd /opt
git clone https://github.com/mxe/mxe mxe
cd mxe

# Set build options.

# The JOBS environment variable controls threads to use when building. DO NOT
# use the regular `make -j4` option with MXE as it will slow down the build.
# Alternatively, you can set this in the make command by appending "JOBS=4"
# to the end of command:
echo "JOBS := 4" >> settings.mk

# The MXE_TARGET environment variable builds MinGW-w64 for 32 bit targets.
# Alternatively, you can specify this in the make command by appending
# "MXE_TARGETS=i686-w64-mingw32" to the end of command:
echo "MXE_TARGETS := i686-w64-mingw32.static" >> settings.mk

# If you want to build 64 bit version, use this:
# echo "MXE_TARGETS := x86_64-w64-mingw32.static" >> settings.mk

# Build required packages. The following provide a minimum required to build
# a reasonable mpv binary (though not an absolute minimum).

make gcc ffmpeg libass jpeg lua luajit

# Add MXE binaries to $PATH
export PATH=/opt/mxe/usr/bin/:$PATH

# Build mpv. The target will be used to automatically select the name of the
# build tools involved (e.g. it will use i686-w64-mingw32.static-gcc).

cd ..
git clone https://github.com/mpv-player/mpv.git
cd mpv
meson setup build --crossfile crossfile
meson compile -C build
```

Native compilation with MSYS2
=============================

For Windows developers looking to get started quickly, MSYS2 can be used to
compile mpv natively on a Windows machine. The MSYS2 repositories have binary
packages for most of mpv's dependencies, so the process should only involve
building mpv itself.

To build 64-bit mpv on Windows:

Installing MSYS2
----------------

1. Download an installer from https://www.msys2.org/

   Both the i686 and the x86_64 version of MSYS2 can build 32-bit and 64-bit
   mpv binaries when running on a 64-bit version of Windows, but the x86_64
   version is preferred since the larger address space makes it less prone to
   fork() errors.

2. Start a MinGW-w64 shell (``mingw64.exe``). **Note:** This is different from
   the MSYS2 shell that is started from the final installation dialog. You must
   close that shell and open a new one.

   For a 32-bit build, use ``mingw32.exe``.
   
   For a build on an ARM64 machine, use `clangarm64.exe`

Updating MSYS2
--------------

To prevent errors during post-install, the MSYS2 core runtime must be updated
separately.

**NOTE:** On ARM64 platforms it is required to enable the ``[clangarm64]`` section in ``/etc/pacman.conf`` before performing these pacman commands.

```bash
# Check for core updates. If instructed, close the shell window and reopen it
# before continuing.
pacman -Syu

# Update everything else
pacman -Su
```

Installing mpv dependencies
---------------------------

x86 or x86_64:
```bash
# Install MSYS2 build dependencies and a MinGW-w64 compiler
pacman -S git $MINGW_PACKAGE_PREFIX-{python,pkgconf,gcc,meson}

# Install the most important MinGW-w64 dependencies. libass and lcms2 are also
# pulled in as dependencies of ffmpeg.
pacman -S $MINGW_PACKAGE_PREFIX-{ffmpeg,libjpeg-turbo,luajit}
```

ARM64:
```bash
# Install MSYS2 build dependencies and clang
pacman -S git $MINGW_PACKAGE_PREFIX-{python,pkgconf,clang,meson,cmake}

# Install the most important MinGW-w64 dependencies. libass and lcms2 are also
# pulled in as dependencies of ffmpeg.
pacman -S $MINGW_PACKAGE_PREFIX-{ffmpeg,libjpeg-turbo,spirv-cross,shaderc}
```

**NOTE:** ARM64 does not support LuaJIT currently, and requires the use of D3D instead of GL, hence the variation in packages.


Building mpv
------------

Finally, compile and install mpv. Binaries will be installed to
``/mingw64/bin``, ``/mingw32/bin``, or ``/clangarm64/bin``.

**NOTE:** For ARM64 platforms, in the setup lines, ``-Dgl=disabled`` must be appended.

```bash
meson setup build --prefix=$MSYSTEM_PREFIX
meson compile -C build
```

Or, compile and install both libmpv and mpv:

```bash
meson setup build -Dlibmpv=true --prefix=$MSYSTEM_PREFIX
meson compile -C build
meson install -C build
```

Linking libmpv with MSVC programs
---------------------------------

mpv/libmpv cannot be built with Visual Studio (Microsoft is too incompetent to
support C99/C11 properly and/or hates open source and Linux too much to
seriously do it). But you can build C++ programs in Visual Studio and link them
with a libmpv built with MinGW.

To do this, you need a Visual Studio which supports ``stdint.h`` (recent ones do),
and you need to create a import library for the mpv DLL:

```bash
lib /name:mpv-1.dll /out:mpv.lib /MACHINE:X64
```

The string in the ``/name:`` parameter must match the filename of the DLL (this
is simply the filename the MSVC linker will use).

Static linking is not possible.

Running mpv
-----------

If you want to run mpv from the MinGW-w64 shell, you will find the experience
much more pleasant if you use the ``winpty`` utility

```bash
pacman -S winpty
winpty mpv.com ToS-4k-1920.mov
```

If you want to move / copy ``mpv.exe`` and ``mpv.com`` to somewhere other than
``/mingw64/bin/`` for use outside the MinGW-w64 shell, they will still depend on
DLLs in that folder. The simplest solution is to add ``C:\msys64\mingw64\bin``
to the windows system ``%PATH%``. Beware though that this can cause problems or
confusion in Cygwin if that is also installed on the machine.

Use of the ANGLE OpenGL backend requires a copy of the D3D compiler DLL that
matches the version of the D3D SDK that ANGLE was built with
(``d3dcompiler_43.dll`` in case of MinGW-built ANGLE) in the path or in the
same folder as mpv. It must be of the same architecture (x86_64 / i686) as the
mpv you compiled.
