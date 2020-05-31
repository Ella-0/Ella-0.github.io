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

```sh
git clone "https://github.com/google/kati" --depth=1
cd kati
ckati
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
ckati install # Not tested
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

For the second build for some weird reason cmake can only be executed with the ful path
therefore we have to type `/usr/bin/cmake` instead of `cmake`. We also need to specify
the make program a cmake is designed to detect Ninja not Samurai. It also can't detect the
c compiler so we pass that manually. And still keep openssl off for now.

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
	-DLLVM_ENABLE_ZLIB=ON \
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

### Net BSD Curses
Some LLVM utilites depend on GNU Ncurses however NCurses is GNU software so we can't use it. We
must use Net BSD Curses instead.

### Final Package List

 - Linux
 - Musl
 - Busybox
 - Toybox
 - Net BSD Curses
 - LLVM
 - Python (for building some LLVM stuff)

### Future Stuff to make it usable

 - doas
 - rustup
 - sway
 - A package manager (probably apk or something simpler)
 
### Some cool replacement utilities

- bat (cat with wings)
- exa (alternative ls)
- dust (alternative dust)
- [samurai](https://github.com/michaelforney/samurai) for self hosting in the future

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
 - cmake (should work though bootstrapping may be difficult; may need to cross compile initially)
 - llvm (requires python with libffi)
 - netbsd-curses (if build kati)
 - musl (if build kati)
 - python
 - kati

## Packages that can't
 - linux (due to complex makefiles (kconfig))
 - busybox (due to complex makefiles (kconfig))
 - toybox (due to complex makefiles (kconfig))
 - libffi (due to autotools deps)
 
## Usefull Resources
 - [ngtc](https://github.com/tpimh/ngtc)
