blink
=====

[![Build Status](https://ci.appveyor.com/api/projects/status/github/crosire/blink?svg=true)](https://ci.appveyor.com/project/crosire/blink)

A tool that lets you edit the source code of a Windows C++ application live, while it is running, without having to go through the usual compile-and-restart cycle. It is effectivly a runtime linker that detects changes to source code files, compiles them and links them back into the running application, while patching all updated functions to jump to the new code.

In contrast to similar projects, **blink does not require you to modify your C++ project in any way**. In fact it works on all x86 and x86-64 applications as long as they were compiled with debug symbols (so that a PDB file is created) and you have their source code somewhere on your system.

![Demo](https://i.imgur.com/sUu3asj.gif)

There now is a library which implements the same concept as blink but for Unix based systems: [jet-live](https://github.com/ddovod/jet-live)

## Build

There are no dependencies, blink is fully standalone (apart from the Windows SDK). You only need a C++17 compliant version of MSVC 2017. Then just build the provided Visual Studio solution.

A quick overview of what each of the source code files contain:

|File                                       |Description                                                                    |
|-------------------------------------------|-------------------------------------------------------------------------------|
|[blink.cpp](source/blink.cpp)              |Application logic, filesystem watcher, main loop                               |
|[blink_linker.cpp](source/blink_linker.cpp)|COFF loader and linker, symbol resolving, function patching                    |
|[main.cpp](source/main.cpp)                |Main entry point, remote thread injection, console output loop                 |
|[coff_reader.cpp](source/coff_reader.cpp)  |Abstraction around a COFF file to deal with extended and non-extended OBJ files|
|[msf_reader.cpp](source/msf_reader.cpp)    |Parser for the Microsoft MSF file format (which PDB files are based on)        |
|[pdb_reader.cpp](source/pdb_reader.cpp)    |Parser for the Microsoft PDB file format                                       |

## Usage

Once you build blink, there are two ways to use it:
1) Either launch your application via blink:\
	```blink.exe foo.exe -arguments```
2) Or attach blink to an already running application:\
	```blink.exe PID``` where PID is the process ID to attach to

Now simply do changes to any of the source code files of your application and see how they are reflected right away. Keep in mind that since blink patches on a function-level, you need a function that is repeatedly called, since changes are only made visible the next time a function is entered.

Optionally, if you define the following functions in your application, blink will call them before and after linking, which can be used to synchronize or save/restore application state:
```c++
extern "C" void __blink_sync(const char *source_file);
extern "C" void __blink_release(const char *source_file, bool success);
```

If blink has trouble finding the right compilation command for your project, make sure to check your build is generating PDB files (Program database in the build options), and prefer to use the `/ZI` ("edit and continue mode") switch over `/Z7` and `/Zi`

## Concept

Here is a quick run-down on how blink works:

When you attach it to an application it will start by finding the path to the matching PDB (program debug database) file generated by the compiler. This information is stored in the executable headers. After finding said file it parses it to obtain the list of source code files that were used to build the application and initialize the symbol table with all internal and external symbols of the application. Then it figures out the common parent path of all user source code files and starts listening for file modifications on that path.

Every time a change is registered, blink looks for the compilation command that was used to compile that source file in the application debug information, executes it to get a new object file reflecting the change and then links that object file into the application process.

The linking for the most part works like a traditional linker would, except that it also replicates some of what the Windows loader would do when you launch an executable, so that it can perform at runtime, rather than at compile-time. It parses the object file (which is in COFF format) and allocates and copies all necessary sections into the application process memory space. It then finds the symbol table of the object file and resolves all of them using symbol information that was previously extracted from the PDB file. The state of global variables is preserved by using the address of the existing symbols, so that the new code references the existing memory, rather than new one. Afterwards the relocation table of the object file is used to adjust all address references in memory to point to the correct addresses. And finally, blink writes a x86 jump instruction to the beginning of all functions in the application that are also part of the new code that was just loaded. This jump targets the new functions, so that every time the old function is called, the call is immediately redirected to the new code.

## Contributing

Any contributions to the project are welcomed, it's recommended to use GitHub [pull requests](https://help.github.com/articles/using-pull-requests/).

## License

All source code in this repository is licensed under a [BSD 2-clause license](LICENSE.md).
