+++
title = "generating backtraces with gdb"
description = "A short guide to generating backtraces with gdb"

[extra]
show_toc = true
+++

Generating a backtrace when debugging a segmentation fault (also known as a segfault or `SIGSEGV`)
is very useful, as it can help pinpoint the location of the bug.

## Step 1: Obtain debug information

First, ensure you have debug information for the program or library.

On Void Linux, this can be done by installing `void-repo-debug` and `xtools`,
and using the [`xdbg(1)`](https://man.voidlinux.org/xdbg.1) tool:

```
# xdbg <package> | xargs -o xbps-install -SA
```

Once you are done debugging, you can remove these packages with:

```
# xdbg <package> | xargs -o xbps-remove -R
```

If you compiled the program yourself, your compiler or the program's build system may have a flag
for enabling debug information.

## Step 2: Generate a backtrace

[`gdb(1)`](https://man.voidlinux.org/gdb.1) will be used to generate the backtrace.
There are two methods that can be used.
If the segmentation fault message includes the phrase "(core dumped)", then a coredump file should exist,
and you should use [method 2](#method-2-inspecting-a-coredump-file-in-gdb).

### Method 1: Running the program in GDB

First, launch `gdb`:

```
$ gdb -q <program>
Reading symbols from <program>...
Reading symbols from /usr/lib/debug/usr/bin/<program>...
(gdb)
```

Then, use the `gdb` command `run`, which can pass command-line arguments to the program as `<args>`:

```
(gdb) run <args>
Starting program: <program>
...
Thread ... received signal SIGSEGV, Segmentation fault.
```

Once the program crashes, go to [step 3](#step-3-print-the-backtrace).

### Method 2: Inspecting a coredump file in GDB

The coredump file is probably in the home directory (`/home/<username>`) or the current working directory,
and is probably named `core.<process id>`.
The [`file(1)`](https://man.voidlinux.org/file.1) utility can be used to inspect coredump files and determine
what process they are from:

```
$ file core.<process id>
core.<process id>: ELF 64-bit LSB core file, x86-64, version 1 (SYSV), SVR4-style, from '<program>', real uid: 1000, effective uid: 1000, real gid: 1000, effective gid: 1000, execfn: '/usr/bin/<program>', platform: 'x86_64'
```

If you do not have "(core dumped)" in the segmentation fault message, you can enable them by raising the coredump limit:

```
$ ulimit -c unlimited
```

Then run the program:

```
$ <program> <args>
Segmentation fault (core dumped)  <program>
```

Find the coredump file (check the current directory and your home directory) and run gdb with it:

```
$ gdb -q <program> -c <coredump>
Reading symbols from prusa-slicer...
Reading symbols from /usr/lib/debug//usr/bin/prusa-slicer...
Core was generated by `prusa-slicer'.
Program terminated with signal SIGSEGV, Segmentation fault.
(gdb)
```

## Step 3: Print the backtrace

Print the backtrace with `bt`:

```c++
(gdb) bt
#0  0x00007f8d49fa0475 in std::__cxx11::basic_string<wchar_t, std::char_traits<wchar_t>, std::allocator<wchar_t> >::_M_data (
    this=<optimized out>) at ./build/x86_64-unknown-linux-gnu/libstdc++-v3/include/bits/basic_string.h:223
#1  std::__cxx11::basic_string<wchar_t, std::char_traits<wchar_t>, std::allocator<wchar_t> >::_M_is_local (this=<optimized out>)
    at ./build/x86_64-unknown-linux-gnu/libstdc++-v3/include/bits/basic_string.h:264
#2  std::__cxx11::basic_string<wchar_t, std::char_traits<wchar_t>, std::allocator<wchar_t> >::capacity (this=<optimized out>)
    at ./build/x86_64-unknown-linux-gnu/libstdc++-v3/include/bits/basic_string.h:1159
#3  std::__cxx11::basic_string<wchar_t, std::char_traits<wchar_t>, std::allocator<wchar_t> >::_M_assign (this=this@entry=0x0, __str=L"")
    at ./build/x86_64-unknown-linux-gnu/libstdc++-v3/include/bits/basic_string.tcc:279
#4  0x00007f8d461a935a in std::__cxx11::basic_string<wchar_t, std::char_traits<wchar_t>, std::allocator<wchar_t> >::assign (__str=L"", 
    this=0x0) at /usr/include/c++/13.2/bits/basic_string.h:1565
#5  0x00007f8d461a93c6 in wxTranslations::SetLanguage (this=0x0, lang=lang@entry=wxLANGUAGE_DEFAULT)
    at /builddir/wxWidgets-gtk3-3.2.5/src/common/translation.cpp:1384
#6  0x000056012ee8053c in Slic3r::GUI::GUI_App::load_language (this=this@entry=0x56014dee2a60, language=..., initial=initial@entry=true)
    at src/slic3r/GUI/GUI_App.cpp:2080
#7  0x000056012ee8163c in Slic3r::GUI::GUI_App::on_init_inner (this=0x56014dee2a60) at src/slic3r/GUI/GUI_App.cpp:1094
#8  0x000056012ee83cf6 in Slic3r::GUI::GUI_App::OnInit (this=<optimized out>) at src/slic3r/GUI/GUI_App.cpp:1036
#9  0x00007f8d46156a23 in wxAppConsoleBase::CallOnInit (this=<optimized out>) at /builddir/wxWidgets-gtk3-3.2.5/include/wx/app.h:93
#10 wxEntry (argc=@0x7f8d462f7444: 1, argv=<optimized out>) at /builddir/wxWidgets-gtk3-3.2.5/src/common/init.cpp:481
#11 0x00007f8d46157492 in wxEntry (argc=<optimized out>, argv=<optimized out>) at /builddir/wxWidgets-gtk3-3.2.5/src/common/init.cpp:509
#12 0x000056012ee6b60b in Slic3r::GUI::GUI_Run (params=...) at src/slic3r/GUI/GUI_Init.cpp:54
#13 0x000056012e82c8c4 in Slic3r::CLI::run (this=this@entry=0x7fff1d0a67a0, argc=1, argv=0x7fff1d0a69d8) at src/PrusaSlicer.cpp:618
#14 0x000056012e8126b4 in main (argc=<optimized out>, argv=<optimized out>) at src/PrusaSlicer.cpp:844
```

*(This example backtrace comes from `prusa-slicer`)*

If you want to print the backtrace to a file, the command is slightly different:
```c++
(gdb) | bt | tee <filename>
#0  0x00007f8d49fa0475 in std::__cxx11::basic_string<wchar_t, std::char_traits<wchar_t>, std::allocator<wchar_t> >::_M_data (
    this=<optimized out>) at ./build/x86_64-unknown-linux-gnu/libstdc++-v3/include/bits/basic_string.h:223
...
```

Include the backtrace you generated in your bug report.