# Creating a Self Hosting Linux Distro with no GNU Components

## The C Library
The core part of any UNIX system is the C library. Most Linux distros use the GNU C Library however, 
as we are creating a Linux distro with no GNU Components we can't use the GNU C Library. 
A good alternative is the MUSL C library.

## The C Compiler
The C Compiler is needed to make the system self hosting. It will be used for compiling Linux, 
the MUSL C LIbrary and any other dependencies. We could use the Tiny C Compiler as it has very 
few dependencies however I was unable to successfully compile Linux or MUSL with TCC therefore 
I had to look elsewhere. LLVM/Clang is a good option but it is large and difficult to compile 
so I decided to go with that.

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

### Net BSD Curses
Some LLVM utilites depend on GNU Ncurses however Ncurses is GNU software so we can't use it. We
must use Net BSD Curses instead.

### LLD


### Clang Frontend

