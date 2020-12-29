# Creating a Self Hosting Linux Distro with no GNU Components

_**NOTE:**_ It is not currently possible to create a self hosting linux distro without any GNU components
as I am yet to find a GNU compatible implementation of make. Yes bmake exists however it is _**NOT**_
compatible with the GNU Make extensions. This means that the system can not compile the Linux kernel
yet I'm hoping, in the future, that either the toybox project or the LLVM project or someone else will create a GNU compatible make so it can be self hosting. I will also move over from using the musl libc to the LLVM libc in the future. I'm currently investigating the use of Google's [Kati](https://github.com/google/kati) as the GNU make alternative. However
it still can not compile the Linux kernel.

## Kati
Kati can be build with itself or make
```sh
git clone "https://github.com/google/kati" --depth=1
cd kati
make
```
### Self host
Note this only works if `git` is installed.
.
```sh
git clone "https://github.com/google/kati" --depth=1
cd kati
CXX=/usr/bin/c++ ckati
doas install -Dm755 ckati /usr/bin
```

### Gettext
get text is a set of GNU tools so we'll have to find an alternative. Luckily sabotage linux already has one called gettext-tiny.

```sh
cd gettext-tiny
ckati
doas ckati install perfix=/usr
```

## The C Library
The core part of any UNIX system is the C library. Most Linux distros use the GNU C Library however, 
as we are creating a Linux distro with no GNU Components we can't use the GNU C Library. 
A good alternative is the MUSL C library.

```sh
wget "https://musl.libc.org/releases/musl-1.2.0.tar.gz"
tar -xf musl-1.2.0.tar.gz
cd musl-1.2.0
./configure --prefix=/usr
make -j8
make DESTDIR=$NEW_ROOT install
```

### Self host

```sh
wget "https://musl.libc.org/releases/musl-1.2.0.tar.gz"
tar -xf musl-1.2.0.tar.gz
./configure --prefix=/usr
ckati
ckati install
```

## CMake

The first time we build CMake we need to dissable `ccmake` to stop it from linking against
ncurses that is included with Abyss Linux. We also need to dissable openssl otherwise cmake
will try to link against openssl which we have not compiled for our target system.

```sh
cmake -G Ninja \
	-DBUILD_CursesDialog=OFF \
	-DCMAKE_BUILD_TYPE=Release \
	-DCMAKE_INSTALL_PREFIX=/usr \
	-DCMAKE_USE_OPENSSL=OFF \
	../
```

For the second build for some weird reason cmake can only be executed with the full path
therefore we have to type `/usr/bin/cmake` instead of `cmake`. We also need to specify
the make program a cmake is designed to detect Ninja not Samurai. It also can't detect the
c compiler so we pass that manually. And still keep openssl off for now.

This can be fixed by setting the `PATH` environment variable.

```sh
/usr/bin/cmake -G Ninja \
	-DCMAKE_MAKE_PROGRAM=/usr/bin/samu \
	-DCMAKE_C_COMPILER=/usr/bin/clang \
	-DCMAKE_CXX_COMPILER=/usr/bin/clang++ \
	-DCMAKE_BUILD_TYPE=Release \
	-DCMAKE_INSTALL_PREFIX=/usr \
	-DCMAKE_USE_OPENSSL=OFF \
	-DBUILD_CursesDialog=ON \
	../
```

## The C Compiler
The C Compiler is needed to make the system self hosting. It will be used for compiling Linux, 
the MUSL C LIbrary and any other dependencies. We could use the Tiny C Compiler as it has very 
few dependencies however I was unable to successfully compile Linux or MUSL with TCC therefore 
I had to look elsewhere. LLVM/Clang is a good option but it is large and difficult to compile 
so I decided to go with that.

## The Core Utilities
For the core utilites we have 2 main options: toybox and busybox. Toybox would be the prefered 
option as it is newer however toybox doesn't have everything we need so I decided to use 
busybox. I do intened to use toybos in the future as it has a newer and thus cleaner code base.

```sh
wget "https://busybox.net/downloads/busybox-1.31.1.tar.bz2"
tar -xf busybox-1.31.1.tar.bz2
cd busybox-1.31.1
ln -s $NEW_ROOT _install
make defconfig
make install -j8
```

### Build Tools
Inoder to build LLVM we need cmake too generate the build files and a build tool. 
We could use make however cmake doesn't create make files compatible with BSD make 
therefore we will be using Ninja as our build tool.

### Compiler Runtime
On most Linux distrabutions libgcc is used as the compiler runtime however this is GNU software 
so we can't use it. Instead we will be using LLVM's compiler-rt.

### Unwinder
Libgcc comes with a buildin unwinder library however, since we aren't using libgcc we need a 
different library as out unwinder. Fortunately we can use LLVM's libunwind.

### C++ Library
Most Linux distros come with libstdc++ as the C++ standard library however libstdc++ is GNU 
software so it can't be used. Fortunately LLVM has it's own C++ library that we can use
instead

### LLVM
LLVM comes with some other usefull tools such as it's own implementations of binutils and 
some core libraries needed by compiler frontends such as clang.

#### Building

```sh
cmake -G Ninja -Wno-dev \
	-DCMAKE_C_COMPILER=/usr/bin/clang \
	-DCMAKE_CXX_COMPILER=/usr/bin/clang++ \
	-DCMAKE_C_COMPILER_TARGET=x86_64-pc-linux-musl \
	-DCMAKE_CXX_COMPILER_TARGET=x86_64-pc-linux-musl \
	-DCMAKE_INSTALL_PREFIX=/usr \
	-DCMAKE_BUILD_TYPE=Release \
	-DLLVM_VERSION_SUFFIX="" \
	-DLLVM_APPEND_VC_REV=OFF \
	-DLLVM_ENABLE_PROJECTS="libunwind;libcxxabi;libcxx;compiler-rt;llvm;lld;clang" \
	-DLLVM_ENABLE_LLD=ON \
	-DLLVM_TARGETS_TO_BUILD="X86" \
	-DLLVM_INSTALL_BINUTILS_SYMLINKS=ON \
	-DLLVM_INSTALL_CCTOOLS_SYMLINKS=ON \
	-DLLVM_INCLUDE_EXAMPLES=OFF \
	-DLLVM_ENABLE_PIC=ON \
	-DLLVM_ENABLE_LTO=OFF \
	-DLLVM_INCLUDE_GO_TESTS=OFF \
	-DLLVM_INCLUDE_TESTS=OFF \
	-DLLVM_HOST_TRIPLE=x86_64-pc-linux-musl \
	-DLLVM_DEFAULT_TARGET_TRIPLE=x86_64-pc-linux-musl \
	-DLLVM_ENABLE_LIBXML2=OFF \
	-DLLVM_ENABLE_ZLIB=YES\
	-DLLVM_BUILD_LLVM_DYLIB=ON \
	-DLLVM_LINK_LLVM_DYLIB=ON \
	-DLLVM_OPTIMIZED_TABLEGEN=ON \
	-DLLVM_INCLUDE_BENCHMARKS=OFF \
	-DLLVM_INCLUDE_DOCS=OFF \
	-DLLVM_TOOL_LLVM_ITANIUM_DEMANGLE_FUZZER_BUILD=OFF \
	-DLLVM_TOOL_LLVM_MC_ASSEMBLE_FUZZER_BUILD=OFF \
	-DLLVM_TOOL_LLVM_MC_DISASSEMBLE_FUZZER_BUILD=OFF \
	-DLLVM_TOOL_LLVM_OPT_FUZZER_BUILD=OFF \
	-DLLVM_TOOL_LLVM_MICROSOFT_DEMANGLE_FUZZER_BUILD=OFF \
	-DLLVM_TOOL_LLVM_GO_BUILD=OFF \
	-DLLVM_INSTALL_UTILS=ON \
	-DLLVM_ENABLE_LIBCXX=ON \
	-DLLVM_STATIC_LINK_CXX_STDLIB=ON \
	-DLLVM_ENABLE_LIBEDIT=OFF \
	-DLLVM_ENABLE_TERMINFO=OFF \
	-DLIBCXX_ENABLE_FILESYSTEM=ON \
	-DLIBCXX_USE_COMPILER_RT=ON \
	-DLIBCXX_HAS_MUSL_LIBC=ON \
	-DLIBCXX_ENABLE_STATIC_ABI_LIBRARY=ON \
	-DLIBCXX_STATICALLY_LINK_ABI_IN_SHARED_LIBRARY=ON \
	-DLIBCXX_STATICALLY_LINK_ABI_IN_STATIC_LIBRARY=ON \
	-DLIBCXX_INSTALL_LIBRARY=OFF \
	-DLIBCXXABI_ENABLE_ASSERTIONS=ON \
	-DLIBCXXABI_USE_COMPILER_RT=ON \
	-DLIBCXXABI_USE_LLVM_UNWINDER=ON \
	-DLIBCXXABI_ENABLE_STATIC_UNWINDER=ON \
	-DLIBCXXABI_STATICALLY_LINK_UNWINDER_IN_SHARED_LIBRARY=YES \
	-DLIBCXXABI_ENABLE_SHARED=OFF \
	-DLIBCXXABI_ENABLE_STATIC=ON \
	-DLIBCXXABI_INSTALL_LIBRARY=OFF \
	-DLIBUNWIND_ENABLE_SHARED=OFF \
	-DLIBUNWIND_ENABLE_STATIC=ON \
	-DLIBUNWIND_INSTALL_LIBRARY=OFF \
	-DLIBUNWIND_USE_COMPILER_RT=ON \
 	-DCLANG_DEFAULT_LINKER=lld \
	-DCLANG_DEFAULT_CXX_STDLIB='libc++' \
	-DCLANG_DEFAULT_RTLIB=compiler-rt \
	-DCLANG_DEFAULT_UNWINDLIB=libunwind \
	-DCLANG_VENDOR="LazyBox" \
	-DCLANG_ENABLE_STATIC_ANALYZER=OFF \
	-DCLANG_ENABLE_ARCMT=OFF \
	-DCLANG_LINK_CLANG_DYLIB=OFF \
	-DCOMPILER_RT_USE_BUILTINS_LIBRARY=ON \
	-DCOMPILER_RT_DEFAULT_TARGET_ONLY=OFF \
	-DCOMPILER_RT_INCLUDE_TESTS=OFF \
	-DCOMPILER_RT_BUILD_SANITIZERS=OFF \
	-DCOMPILER_RT_BUILD_XRAY=OFF \
	-DCOMPILER_RT_INCLUDE_TESTS=OFF \
	-DCOMPILER_RT_BUILD_LIBFUZZER=OFF \
	-DENABLE_EXPERIMENTAL_NEW_PASS_MANAGER=TRUE \
	../llvm
samu
env DESTDIR=/path/to/your/install/root samu install
```

### ZLib

```sh
./configure --prefix=/usr
ckati
doas ckati install
```

### Berkley Yacc
Berkley Yacc is an alternative implementation of Yacc (Yet Another Compiler Compiler) that i'll use since
GNU Bison is offlimits.

```sh
./configure --prefix=/usr
bmake
doas bmake install
```

### Net BSD Curses
Some LLVM utilites depend on GNU Ncurses however NCurses is GNU software so we can't use it. We
must use Net BSD Curses instead.

```sh
ckati
ckati PREFIX=/usr install
```

### Doas
Since sudo is massive, I've chosen to go with a port of OpenBSD doas on this distro for authenticating as
a different user with your own password. _**NOTE**_ byacc is needed to build doas.

```sh
yacc  parse.y
mv -f y.tab.c parse.c
cc -I. -I./libopenbsd -Wall -Wextra -Werror -pedantic -MD -MP -Wno-unused-result -D__linux__ -D_DEFAULT_SOURCE -D_GNU_SOURCE -DUID_MAX=65535 -DGID_MAX=65535 -DHAVE_EXPLICIT_BZERO -DHAVE_STRLCAT -DHAVE_STRLCPY -DHAVE_EXECVPE -DHAVE_SETRESUID -DHAVE_SYSCONF -DHAVE_PROC_PID -DHAVE_DIRFD -DHAVE_FCNTL_H -DHAVE_DIRENT_H -DHAVE_SYS_DIR_H -DHAVE___ATTRIBUTE__ -DHAVE_SHADOW_H -DUSE_SHADOW  -c parse.c -o parse.o
cc -I. -I./libopenbsd -Wall -Wextra -Werror -pedantic -MD -MP -Wno-unused-result -D__linux__ -D_DEFAULT_SOURCE -D_GNU_SOURCE -DUID_MAX=65535 -DGID_MAX=65535 -DHAVE_EXPLICIT_BZERO -DHAVE_STRLCAT -DHAVE_STRLCPY -DHAVE_EXECVPE -DHAVE_SETRESUID -DHAVE_SYSCONF -DHAVE_PROC_PID -DHAVE_DIRFD -DHAVE_FCNTL_H -DHAVE_DIRENT_H -DHAVE_SYS_DIR_H -DHAVE___ATTRIBUTE__ -DHAVE_SHADOW_H -DUSE_SHADOW  -c doas.c -o doas.o
cc -I. -I./libopenbsd -Wall -Wextra -Werror -pedantic -MD -MP -Wno-unused-result -D__linux__ -D_DEFAULT_SOURCE -D_GNU_SOURCE -DUID_MAX=65535 -DGID_MAX=65535 -DHAVE_EXPLICIT_BZERO -DHAVE_STRLCAT -DHAVE_STRLCPY -DHAVE_EXECVPE -DHAVE_SETRESUID -DHAVE_SYSCONF -DHAVE_PROC_PID -DHAVE_DIRFD -DHAVE_FCNTL_H -DHAVE_DIRENT_H -DHAVE_SYS_DIR_H -DHAVE___ATTRIBUTE__ -DHAVE_SHADOW_H -DUSE_SHADOW  -c env.c -o env.o
cc -I. -I./libopenbsd -Wall -Wextra -Werror -pedantic -MD -MP -Wno-unused-result -D__linux__ -D_DEFAULT_SOURCE -D_GNU_SOURCE -DUID_MAX=65535 -DGID_MAX=65535 -DHAVE_EXPLICIT_BZERO -DHAVE_STRLCAT -DHAVE_STRLCPY -DHAVE_EXECVPE -DHAVE_SETRESUID -DHAVE_SYSCONF -DHAVE_PROC_PID -DHAVE_DIRFD -DHAVE_FCNTL_H -DHAVE_DIRENT_H -DHAVE_SYS_DIR_H -DHAVE___ATTRIBUTE__ -DHAVE_SHADOW_H -DUSE_SHADOW  -c shadow.c -o shadow.o
cc -I. -I./libopenbsd -Wall -Wextra -Werror -pedantic -MD -MP -Wno-unused-result -D__linux__ -D_DEFAULT_SOURCE -D_GNU_SOURCE -DUID_MAX=65535 -DGID_MAX=65535 -DHAVE_EXPLICIT_BZERO -DHAVE_STRLCAT -DHAVE_STRLCPY -DHAVE_EXECVPE -DHAVE_SETRESUID -DHAVE_SYSCONF -DHAVE_PROC_PID -DHAVE_DIRFD -DHAVE_FCNTL_H -DHAVE_DIRENT_H -DHAVE_SYS_DIR_H -DHAVE___ATTRIBUTE__ -DHAVE_SHADOW_H -DUSE_SHADOW  -c libopenbsd/errc.c -o libopenbsd/errc.o
cc -I. -I./libopenbsd -Wall -Wextra -Werror -pedantic -MD -MP -Wno-unused-result -D__linux__ -D_DEFAULT_SOURCE -D_GNU_SOURCE -DUID_MAX=65535 -DGID_MAX=65535 -DHAVE_EXPLICIT_BZERO -DHAVE_STRLCAT -DHAVE_STRLCPY -DHAVE_EXECVPE -DHAVE_SETRESUID -DHAVE_SYSCONF -DHAVE_PROC_PID -DHAVE_DIRFD -DHAVE_FCNTL_H -DHAVE_DIRENT_H -DHAVE_SYS_DIR_H -DHAVE___ATTRIBUTE__ -DHAVE_SHADOW_H -DUSE_SHADOW  -c libopenbsd/verrc.c -o libopenbsd/verrc.o
cc -I. -I./libopenbsd -Wall -Wextra -Werror -pedantic -MD -MP -Wno-unused-result -D__linux__ -D_DEFAULT_SOURCE -D_GNU_SOURCE -DUID_MAX=65535 -DGID_MAX=65535 -DHAVE_EXPLICIT_BZERO -DHAVE_STRLCAT -DHAVE_STRLCPY -DHAVE_EXECVPE -DHAVE_SETRESUID -DHAVE_SYSCONF -DHAVE_PROC_PID -DHAVE_DIRFD -DHAVE_FCNTL_H -DHAVE_DIRENT_H -DHAVE_SYS_DIR_H -DHAVE___ATTRIBUTE__ -DHAVE_SHADOW_H -DUSE_SHADOW  -c libopenbsd/progname.c -o libopenbsd/progname.o
cc -I. -I./libopenbsd -Wall -Wextra -Werror -pedantic -MD -MP -Wno-unused-result -D__linux__ -D_DEFAULT_SOURCE -D_GNU_SOURCE -DUID_MAX=65535 -DGID_MAX=65535 -DHAVE_EXPLICIT_BZERO -DHAVE_STRLCAT -DHAVE_STRLCPY -DHAVE_EXECVPE -DHAVE_SETRESUID -DHAVE_SYSCONF -DHAVE_PROC_PID -DHAVE_DIRFD -DHAVE_FCNTL_H -DHAVE_DIRENT_H -DHAVE_SYS_DIR_H -DHAVE___ATTRIBUTE__ -DHAVE_SHADOW_H -DUSE_SHADOW  -c libopenbsd/readpassphrase.c -o libopenbsd/readpassphrase.o
cc -I. -I./libopenbsd -Wall -Wextra -Werror -pedantic -MD -MP -Wno-unused-result -D__linux__ -D_DEFAULT_SOURCE -D_GNU_SOURCE -DUID_MAX=65535 -DGID_MAX=65535 -DHAVE_EXPLICIT_BZERO -DHAVE_STRLCAT -DHAVE_STRLCPY -DHAVE_EXECVPE -DHAVE_SETRESUID -DHAVE_SYSCONF -DHAVE_PROC_PID -DHAVE_DIRFD -DHAVE_FCNTL_H -DHAVE_DIRENT_H -DHAVE_SYS_DIR_H -DHAVE___ATTRIBUTE__ -DHAVE_SHADOW_H -DUSE_SHADOW  -c libopenbsd/strtonum.c -o libopenbsd/strtonum.o
cc -I. -I./libopenbsd -Wall -Wextra -Werror -pedantic -MD -MP -Wno-unused-result -D__linux__ -D_DEFAULT_SOURCE -D_GNU_SOURCE -DUID_MAX=65535 -DGID_MAX=65535 -DHAVE_EXPLICIT_BZERO -DHAVE_STRLCAT -DHAVE_STRLCPY -DHAVE_EXECVPE -DHAVE_SETRESUID -DHAVE_SYSCONF -DHAVE_PROC_PID -DHAVE_DIRFD -DHAVE_FCNTL_H -DHAVE_DIRENT_H -DHAVE_SYS_DIR_H -DHAVE___ATTRIBUTE__ -DHAVE_SHADOW_H -DUSE_SHADOW  -c libopenbsd/reallocarray.c -o libopenbsd/reallocarray.o
cc -I. -I./libopenbsd -Wall -Wextra -Werror -pedantic -MD -MP -Wno-unused-result -D__linux__ -D_DEFAULT_SOURCE -D_GNU_SOURCE -DUID_MAX=65535 -DGID_MAX=65535 -DHAVE_EXPLICIT_BZERO -DHAVE_STRLCAT -DHAVE_STRLCPY -DHAVE_EXECVPE -DHAVE_SETRESUID -DHAVE_SYSCONF -DHAVE_PROC_PID -DHAVE_DIRFD -DHAVE_FCNTL_H -DHAVE_DIRENT_H -DHAVE_SYS_DIR_H -DHAVE___ATTRIBUTE__ -DHAVE_SHADOW_H -DUSE_SHADOW  -c libopenbsd/closefrom.c -o libopenbsd/closefrom.o
ar -r libopenbsd.a libopenbsd/errc.o libopenbsd/verrc.o libopenbsd/progname.o libopenbsd/readpassphrase.o libopenbsd/strtonum.o libopenbsd/reallocarray.o libopenbsd/closefrom.o
cc -I. -I./libopenbsd -Wall -Wextra -Werror -pedantic -MD -MP -Wno-unused-result -D__linux__ -D_DEFAULT_SOURCE -D_GNU_SOURCE -DUID_MAX=65535 -DGID_MAX=65535 -DHAVE_EXPLICIT_BZERO -DHAVE_STRLCAT -DHAVE_STRLCPY -DHAVE_EXECVPE -DHAVE_SETRESUID -DHAVE_SYSCONF -DHAVE_PROC_PID -DHAVE_DIRFD -DHAVE_FCNTL_H -DHAVE_DIRENT_H -DHAVE_SYS_DIR_H -DHAVE___ATTRIBUTE__ -DHAVE_SHADOW_H parse.o doas.o env.o shadow.o libopenbsd.a -o doas -lcrypt
rm parse.c
```
### Final Package List

 - Linux
 - Musl
 - Busybox
 - Toybox
 - Net BSD Curses
 - LLVM
 - Python (for building some LLVM stuff)

### Future Stuff to make it usable

 - doas - partially working though "can't authenticate error"
 - rustup
 - sway
 - A package manager (probably apk or something simpler)
 
### Some cool replacement utilities

- bat (cat with wings) - works but busybox less doesn't support raw char flag
- exa (alternative ls) - 
- dust (alternative du)
- [samurai](https://github.com/michaelforney/samurai) for self hosting in the future

### Rust
Rust is a weird one as it is very difficult to compile. Rust is usually installed through the rust toolchain installer (rustup)
though the x86_64-unknown-linux-musl toolchain provided links dynamically against libgcc_s which is not provided
in our system for reasons that are farely obvious. In future, upstream should link libgcc or libunwind statically
inorder to work on more obscure musl systems though for now this isn't the case so we need to cross compile our own.

So far I have not been able to successfully build a toolchain with a statically linked unwinder but I can link rust
against the unwinder in our target chroot fairly easily with minimal changes.

The first change to make is with the libunwind crate. In lib.rs we need to tell rust to link against libunwind rather
than libgcc.

```rust
#![no_std]
#![unstable(feature = "panic_unwind", issue = "32837")]
#![feature(link_cfg)]
#![feature(nll)]
#![feature(staged_api)]
#![feature(unwind_attributes)]
#![feature(static_nobundle)]
#![cfg_attr(not(target_env = "msvc"), feature(libc))]

cfg_if::cfg_if! {
    if #[cfg(target_env = "msvc")] {
        // no extra unwinder support needed
    } else if #[cfg(all(target_arch = "wasm32", not(target_os = "emscripten")))] {
        // no unwinder on the system!
    } else {
        mod libunwind;
        pub use libunwind::*;
    }
}

#[cfg(target_env = "musl")]
#[link(name = "unwind", kind = "static", cfg(target_feature = "crt-static"))]
#[link(name = "unwind", cfg(not(target_feature = "crt-static")))] // **THE CHANGED LINE**
extern "C" {}

#[cfg(target_os = "redox")]
#[link(name = "gcc_eh", kind = "static-nobundle", cfg(target_feature = "crt-static"))]
#[link(name = "gcc_s", cfg(not(target_feature = "crt-static")))]
extern "C" {}

#[cfg(all(target_vendor = "fortanix", target_env = "sgx"))]
#[link(name = "unwind", kind = "static-nobundle")]
extern "C" {}
```

Then we need to create a directory called `extras` in `src/ci/docker/` and populate it with
all of the files matching `/usr/lib/libunwind.so*`

Then we need to edit the musl ci docker file to look like this

```docker
FROM ubuntu:16.04

RUN apt-get update && apt-get install -y --no-install-recommends \
  g++ \
  make \
  file \
  wget \
  curl \
  ca-certificates \
  python3 \
  git \
  cmake \
  xz-utils \
  sudo \
  gdb \
  patch \
  libssl-dev \
  libunwind-dev \
  pkg-config

WORKDIR /build/

COPY scripts/musl-toolchain.sh /build/
# We need to mitigate rust-lang/rust#34978 when compiling musl itself as well
RUN CFLAGS="-Wa,-mrelax-relocations=no -Wa,--compress-debug-sections=none -Wl,--compress-debug-sections=none" \
    CXXFLAGS="-Wa,-mrelax-relocations=no -Wa,--compress-debug-sections=none -Wl,--compress-debug-sections=none" \
    REPLACE_CC=1 bash musl-toolchain.sh x86_64 && rm -rf build

COPY scripts/sccache.sh /scripts/
RUN sh /scripts/sccache.sh


COPY extras/libunwind.so /usr/local/x86_64-linux-musl/lib
COPY extras/libunwind.so.1 /usr/local/x86_64-linux-musl/lib
COPY extras/libunwind.so.1.0 /usr/local/x86_64-linux-musl/lib


ENV HOSTS=x86_64-unknown-linux-musl

ENV RUST_CONFIGURE_ARGS \
      --musl-root-x86_64=/usr/local/x86_64-linux-musl \
      --enable-extended \
      --disable-docs \
      --enable-lld \
      --enable-llvm-libunwind \
      --enable-llvm-static-stdcpp \
      --set target.x86_64-unknown-linux-musl.crt-static=false \
      --build $HOSTS

# Newer binutils broke things on some vms/distros (i.e., linking against
# unknown relocs disabled by the following flag), so we need to go out of our
# way to produce "super compatible" binaries.
#
# See: https://github.com/rust-lang/rust/issues/34978
# And: https://github.com/rust-lang/rust/issues/59411
ENV CFLAGS_x86_64_unknown_linux_musl="-Wa,-mrelax-relocations=no -Wa,--compress-debug-sections=none \
    -Wl,--compress-debug-sections=none"

# To run native tests replace `dist` below with `test`
ENV SCRIPT python3 ../x.py dist -i --build $HOSTS
```

then we can do
```
doas src/ci/docker/run.sh dist-x86_64-musl
```
which will build out toolchain and output it in several tarballs in `obj/build/dist`
which then can be copied and extracted in the target system

### Samurai

#### Boot-strap

To boot-strap Samurai start by running the following which will compile the whole program manualy

```
$ cc -o samu build.c deps.c env.c graph.c htab.c log.c parse.c samu.c scan.c tool.c tree.c util.c
```

#### Recompile

Once this has run we can now run `./samu` which will run the `build.ninja` file that's in that directory
thus compiling Samurai whith itself.

#### Install

Now we need to install Samurai which can be done with `install -D -m 755 samu /usr/bin/samu`

## Packages that can be selfhosted
 - samurai (build manually then with its self)
 - cmake
 - llvm (requires python with libffi)
 - netbsd-curses (if build kati)
 - musl (if build kati)
 - python
 - kati
 - rust
 - doas
 - byacc

## Packages that can't
 - linux (due to complex makefiles (kconfig))
 - busybox (due to complex makefiles (kconfig))
 - toybox (due to complex makefiles (kconfig))
 - libffi (due to autotools deps)
 
## Usefull Resources
 - [ngtc](https://github.com/tpimh/ngtc)
