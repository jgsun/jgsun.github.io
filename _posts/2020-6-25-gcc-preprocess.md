---
layout: post
title:  "使用编译预处理快速查询宏定义"
categories: build
tags: gcc, preprocess, sysroot
author: jgsun
---


* content
{:toc}

# 1. Overview
今天跟一位资深业务同事定位一个文件读写问题时，想确定宏O_CREAT的定义，我想到的是去查找sysroot头文件，这位同事却使用编译预处理，三下五除二就快速找到了这个宏定义是0x0100，并定位出sysroot头文件的位置， 让我叹为观止，自愧不如，真是"三人行，必有我师"。下面记录了操作过程。
![image](/images/posts/build/gcc-preprocess/macro.png)






















# 2. 查找编译命令
```
jiangusu@OS$ cd /repo/jiangusu/ms/sw/y/build/apps/fsproxy_app/ 
jiangusu@fsproxy_app$ cat fsproxy_app_isam-reborn-cavium-sdmva.nostrip.link.cmd | grep fs_ops.c
cd /repo/jiangusu/ms/sw/vobs/esam/objects/lscx-a/gREBORN_isam-reborn-cavium-sdmva_fsproxy_app_isam-reborn-cavium-sdmva/sources;/repo/jiangusu/ms/buildroot/output/host/usr/bin/mips64-octeon-linux-gnu-gcc -D_BIG_ENDIAN=1 -Werror -O0 -Wall -Wno-write-strings -Wno-psabi -g -DCOMPILER_GNU=730 -DTARG_OS_NONE= -DTARG_ARCH_REBORN= -DTARG_ARCH_= -DBOARD_lscx_a -DLSCX_A_BOARD -fno-strict-aliasing -fdata-sections -ffunction-sections -funwind-tables -fsigned-char -D_GNU_SOURCE -D_LARGEFILE_SOURCE -D_LARGEFILE64_SOURCE -D_FILE_OFFSET_BITS=64 -DCHX_FAMILY -O2 -DORM_APP_NAME=\"fsproxy_app\" -DGOOGLE_PROTOBUF_NO_RTTI -Dfar= -Dnear= -D__STDC_FORMAT_MACROS=1 -Iinclude -Iitf -Iidl -I/repo/jiangusu/ms/sw/vobs/dsl/sw/y/src/fsproxy/inc -Iy/include -Wp,-MD,../vobs/dsl/sw/y/src/fsproxy/E9CDEFE/src/fs_ops.d -o ../vobs/dsl/sw/y/src/fsproxy/E9CDEFE/src/fs_ops.o -c y/src/fsproxy/src/fs_ops.ccd /repo/jiangusu/ms/sw/vobs/esam/objects/lscx-a/gREBORN_isam-reborn-cavium-sdmva_fsproxy_app_isam-reborn-cavium-sdmva/sources;/repo/jiangusu/ms/buildroot/output/host/usr/bin/mips64-octeon-linux-gnu-gcc -D_BIG_ENDIAN=1 -Werror -O0 -Wall -Wno-write-strings -Wno-psabi -g -DCOMPILER_GNU=730 -DTARG_OS_NONE= -DTARG_ARCH_REBORN= -DTARG_ARCH_= -DBOARD_lscx_a -DLSCX_A_BOARD -fno-strict-aliasing -fdata-sections -ffunction-sections -funwind-tables -fsigned-char -D_GNU_SOURCE -D_LARGEFILE_SOURCE -D_LARGEFILE64_SOURCE -D_FILE_OFFSET_BITS=64 -DCHX_FAMILY -O2 -DORM_APP_NAME=\"fsproxy_app\" -DGOOGLE_PROTOBUF_NO_RTTI -Dfar= -Dnear= -D__STDC_FORMAT_MACROS=1 -Iinclude -Iitf -Iidl -I/repo/jiangusu/ms/sw/vobs/dsl/sw/y/src/fsproxy/inc -Iy/include -Wp,-MD,../vobs/dsl/sw/y/src/fsproxy/E9CDEFE/src/fs_ops.d -o ../vobs/dsl/sw/y/src/fsproxy/E9CDEFE/src/fs_ops.o -c y/src/fsproxy/src/fs_ops.c
```
# 3. 修改编译命令，进行预处理编译
```
cd /repo/jiangusu/ms/sw/vobs/esam/objects/lscx-a/gREBORN_isam-reborn-cavium-sdmva_fsproxy_app_isam-reborn-cavium-sdmva/sources;/repo/jiangusu/ms/buildroot/output/host/usr/bin/mips64-octeon-linux-gnu-gcc -D_BIG_ENDIAN=1 -Werror -O0 -Wall -Wno-write-strings -Wno-psabi -g -DCOMPILER_GNU=730 -DTARG_OS_NONE= -DTARG_ARCH_REBORN= -DTARG_ARCH_= -DBOARD_lscx_a -DLSCX_A_BOARD -fno-strict-aliasing -fdata-sections -ffunction-sections -funwind-tables -fsigned-char -D_GNU_SOURCE -D_LARGEFILE_SOURCE -D_LARGEFILE64_SOURCE -D_FILE_OFFSET_BITS=64 -DCHX_FAMILY -O2 -DORM_APP_NAME=\"fsproxy_app\" -DGOOGLE_PROTOBUF_NO_RTTI -Dfar= -Dnear= -D__STDC_FORMAT_MACROS=1 -Iinclude -Iitf -Iidl -I/repo/jiangusu/ms/sw/vobs/dsl/sw/y/src/fsproxy/inc -Iy/include -Wp,-MD,../vobs/dsl/sw/y/src/fsproxy/E9CDEFE/src/fs_ops.d -o ../vobs/dsl/sw/y/src/fsproxy/E9CDEFE/src/fs_ops.o -c y/src/fsproxy/src/fs_ops.ccd /repo/jiangusu/ms/sw/vobs/esam/objects/lscx-a/gREBORN_isam-reborn-cavium-sdmva_fsproxy_app_isam-reborn-cavium-sdmva/sources;/repo/jiangusu/ms/buildroot/output/host/usr/bin/mips64-octeon-linux-gnu-gcc -D_BIG_ENDIAN=1 -Werror -O0 -Wall -Wno-write-strings -Wno-psabi -g -DCOMPILER_GNU=730 -DTARG_OS_NONE= -DTARG_ARCH_REBORN= -DTARG_ARCH_= -DBOARD_lscx_a -DLSCX_A_BOARD -fno-strict-aliasing -fdata-sections -ffunction-sections -funwind-tables -fsigned-char -D_GNU_SOURCE -D_LARGEFILE_SOURCE -D_LARGEFILE64_SOURCE -D_FILE_OFFSET_BITS=64 -DCHX_FAMILY -O2 -DORM_APP_NAME=\"fsproxy_app\" -DGOOGLE_PROTOBUF_NO_RTTI -Dfar= -Dnear= -D__STDC_FORMAT_MACROS=1 -Iinclude -Iitf -Iidl -I/repo/jiangusu/ms/sw/vobs/dsl/sw/y/src/fsproxy/inc -Iy/include -Wp,-MD,../vobs/dsl/sw/y/src/fsproxy/E9CDEFE/src/fs_ops.d -o ../vobs/dsl/sw/y/src/fsproxy/E9CDEFE/src/fs_ops.i -E y/src/fsproxy/src/fs_ops.c
```
对比一下，修改了` -o ../vobs/dsl/sw/y/src/fsproxy/E9CDEFE/src/fs_ops.i -E y/src/fsproxy/src/fs_ops.c`, 将编译参数改为-E。
# 4. 查看宏定义和sysroot头文件
`jiangusu@sources$ vi ../vobs/dsl/sw/y/src/fsproxy/E9CDEFE/src/fs_ops.i`
![image](/images/posts/build/gcc-preprocess/result.png)


![image](/images/posts/build/gcc-preprocess/headfile.png)



# 5. GCC help
GCC help菜单解释-E选项是‘-E Preprocess only; do not compile, assemble or link’
```
jiangusu@~$ gcc --help
Usage: gcc [options] file...
Options:
  -pass-exit-codes Exit with highest error code from a phase
  --help Display this information
  --target-help Display target specific command line options
  --help={common|optimizers|params|target|warnings|[^]{joined|separate|undocumented}}[,...]
                           Display specific types of command line options
  (Use '-v --help' to display command line options of sub-processes)
  --version Display compiler version information
  -dumpspecs Display all of the built in spec strings
  -dumpversion Display the version of the compiler
  -dumpmachine Display the compiler's target processor
  -print-search-dirs Display the directories in the compiler's search path
  -print-libgcc-file-name Display the name of the compiler's companion library
  -print-file-name=<lib> Display the full path to library <lib>
  -print-prog-name=<prog> Display the full path to compiler component <prog>
  -print-multiarch Display the target's normalized GNU triplet, used as
                           a component in the library path
  -print-multi-directory Display the root directory for versions of libgcc
  -print-multi-lib Display the mapping between command line options and
                           multiple library search directories
  -print-multi-os-directory Display the relative path to OS libraries
  -print-sysroot Display the target libraries directory
  -print-sysroot-headers-suffix Display the sysroot suffix used to find headers
  -Wa,<options> Pass comma-separated <options> on to the assembler
  -Wp,<options> Pass comma-separated <options> on to the preprocessor
  -Wl,<options> Pass comma-separated <options> on to the linker
  -Xassembler <arg> Pass <arg> on to the assembler
  -Xpreprocessor <arg> Pass <arg> on to the preprocessor
  -Xlinker <arg> Pass <arg> on to the linker
  -save-temps Do not delete intermediate files
  -save-temps=<arg> Do not delete intermediate files
  -no-canonical-prefixes Do not canonicalize paths when building relative
                           prefixes to other gcc components
  -pipe Use pipes rather than intermediate files
  -time Time the execution of each subprocess
  -specs=<file> Override built-in specs with the contents of <file>
  -std=<standard> Assume that the input sources are for <standard>
  --sysroot=<directory> Use <directory> as the root directory for headers
                           and libraries
  -B <directory> Add <directory> to the compiler's search paths
  -v Display the programs invoked by the compiler
  -### Like -v but options quoted and commands not executed
  -E Preprocess only; do not compile, assemble or link
  -S Compile only; do not assemble or link
  -c Compile and assemble, but do not link
  -o <file> Place the output into <file>
  -pie Create a position independent executable
  -shared Create a shared library
  -x <language> Specify the language of the following input files
                           Permissible languages include: c c++ assembler none
                           'none' means revert to the default behavior of
                           guessing the language based on the file's extension

Options starting with -g, -f, -m, -O, -W, or --param are automatically
 passed on to the various sub-processes invoked by gcc. In order to pass
 other options on to these processes the -W<letter> options must be used.

For bug reporting instructions, please see:
<http://bugzilla.redhat.com/bugzilla>.
```