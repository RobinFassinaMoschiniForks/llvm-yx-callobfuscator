# LLVM-YX-CALLOBFUSCATOR


LLVM plugin to transparently apply stack spoofing and indirect syscalls, if possible, to Windows x64 native calls at compile time.


## "I've 5 mins, what is this?"
This project is a plugin meant to be used with [opt](https://llvm.org/docs/CommandGuide/opt.html), the LLVM optimizer. Opt will use the pass included in this plugin to hook calls to Windows functions based on a config file and point those calls to a single function, which, given an ID identifying the function to be called, will apply dynamic stack obfuscation, and if the function is a syscall stub, will call it throw indirect syscalling.


**Brief**: Set up a config file indicating the functions to be hooked, write your code without caring about Windows function calls, compile with clang to generate .ir files, give them to opt along with this plugin, opt hooks functions, llc compiles the .ir to a.obj, and ld links it to an executable that automatically obfuscates function calls.


**Disclaimer**: This only works for code that you compile. This means that if you are linking against a compiled standard library and that library makes a call to a function, it will not be covered by opt (unless the library has an llvm-ir version, but this is uncommon).


## Table of Contents
> * [Setup](#setup)
> * [Usage and example](#usage-and-example)
> * [Developer guide](#developer-guide)
>   * [File distribution](#file-distribution)
>   * [How the pass works](#how-the-pass-works)
>   * [How the dispatching system works](#how-the-dispatching-system-works)
> * [Thanks](#thanks)
> * [TODO](#todo)

## Setup
This setup is written for Windows, but it should be possible to setup this environment in Linux easily (still output executables can only be built for Windows x64). This was tested with LLVM 16.x and 17.x.


* **Dependencies**:

    To be able to compile this project, we mainly need 2 things: LLVM and CMAKE.

    LLVM can be either compiled from source, downloaded from the LLVM releases, or installed through MSYS2. For a guide on how to compile LLVM from source and compile a basic plugin, [see this](https://github.com/janoglezcampos/llvm-pass-plugin-skeleton?tab=readme-ov-file#llvm-optimization-pass-skeleton). We also need to set up CMAKE and our build tools. Clang is required to compile the helpers library. For the linker, we don't care too much. Lastly, as a generator, I prefer Ninja, but make, nmake or msbuild will work.

    Everything listed above can be installed with pacman by using MSYS2.
  * Download and install [MSYS2](https://www.msys2.org/).
  * Launch an MSYS2 Mingw64 terminal and install the following packages:
  * Install LLVM: ```pacman -S mingw-w64-x86_64-llvm```
  * Install Clang: ```pacman -S mingw-w64-x86_64-clang```
  * Install Cmake: ```pacman -S mingw-w64-x86_64-cmake```
  * Install Ninja: ```pacman -S mingw-w64-x86_64-ninja```

* **Building**:
  
  First, clone this project, pretty obvious:

        git clone https://github.com/janoglezcampos/llvm-yx-callobfuscator


    To build this project, we will be using CMAKE. Because I find it convenient, I use VScode with the CMake extension. You will find my CMAKE config at ```.vscode_conf/setting.json```

    In any other case, launch an MSYS2 Mingw64 terminal and do exactly what I say (without checking what any of the commands Im giving you will do to your beloved machine):

    Go to the root directory of this repo:

        cd <whatever>/llvm-yx-obfuscator

    Create a build directory and change the directory to it:

        mkdir build; cd build

    Choose an install location. I recommend doing this so it will be easier to either get the files in the right place or just be able to pick them up easily. Also, the [usage](#usage-and-example) section will use this folder as the relative folder for accessing these files, so if you add it to the PATH, the commands will work right away. I use  ```C:/Users/<my-user>/llvm-plugins```, but creating a folder in the project directory called ```install```, side by side with ```build``` will do. This folder will not be edited if you don't run the install command, but you will have to go get the files in the build folder.


    Configure the project; here you specify the installation folder. ```DCMAKE_BUILD_TYPE``` will set the default mode: Debug, Release or MinSizeRel. Depending on which generator you are using, you are going to be able to change this later or not. I use Nija as the generator, but you can use any other.

        cmake -G Ninja -DCMAKE_INSTALL_PREFIX="<path_to_install_folder>" -DCMAKE_BUILD_TYPE=Release ./..

    Build the project; optionally choose mode with ```—-config Release``` (if your generator lets you):

        cmake --build .

    Move the generated files to the install directory you chose before. If you don't want to use the install feature of CMake, just get the files (```libCallObfuscatorHelpers.a``` and ```CallObfuscatorPlugin.dll```) from the build directory and put them in a place you remember; you will need them.

        cmake --build . --target install

    At this point, you should have two files:
    * ```libCallObfuscatorHelpers.a```: A C library that includes all the logic that needs to be executed at runtime.  
    * ```CallObfuscatorPlugin.dll```:  The actual plugin, written in C++, that will be compiled and linked to a dll.

## Usage and example


First of all, we need to set up our configuration file. In this section, we will be building the project found in the example folder, so the config file is already made. You can add any number of functions to the file, and the functions do not need to appear in the program.

The plugin will get the path to the config from an environment variable called ```LLVM_OBF_FUNCTIONS```. You can either add it along with all the other environment variables for the user, the system, or just set it up for the current terminal. You can also set it in the makefile, so it is only set for the compilation.

Now is time to run the pass. Remember that there is a makefile already set up inside the example project. To better understand the process shown here, [see this](https://github.com/janoglezcampos/llvm-pass-plugin-skeleton?tab=readme-ov-file#running-you-pass).

* Go inside the example folder and create a build folder; inside, create 2 folders: irs and objs.

        cd example; mkdir build; mkdir build/irs; mkdifr build/objs
* Set ```LLVM_OBF_FUNCTIONS``` to point to the full path of ```callobfuscator.conf```, only for this terminal:

    * MSYS2/Linux: ```export LLVM_OBF_FUNCTIONS=<absolute path to callobfuscator.conf>```
    * Windows PowerShell: ```env:LLVM_OBF_FUNCTIONS=<absolute path to callobfuscator.conf>```
  
* Compile the C files to LLVM-IR:

        clang -O0 -Xclang -disable-O0-optnone -S -emit-llvm ./source/utils.c -Iheaders -o ./build/irs/utils.ll

        clang -O0 -Xclang -disable-O0-optnone -S -emit-llvm ./source/main.c -Iheaders -o ./build/irs/main.ll
* Merge all files:

        llvm-link ./build/irs/main.ll ./build/irs/utils.ll -S -o ./build/irs/example.ll
* Run obfusction pass:

        opt -load-pass-plugin="<path to the pass dll>" -passes="base-plugin-pass" ./build/irs/example.ll -o ./build/irs/example.obf.ll
* Run optimization passes:

        opt -O3 ./build/irs/example.obf.ll -o ./build/irs/example.op.ll
* Compile to windows x86_64 assembly:
  
        llc --mtriple=x86_64-pc-windows-msvc -filetype=obj  ./build/irs/example.op.ll -o ./build/objs/example.obj
* Link:

        clang ./build/objs/example.obj -o ./build/example.exe


Now you should have ```./build/example.exe```, the final executable.

## Developer guide
* ### File distribution
    ---
    The code is always divided into two folders, one called headers, for definitions and macros mainly, and the other called source, containing the actual source   code. For every source code file, there is a header file matching the relative path to the source folder, in the headers folder. Documentation for functions is    always found at headers files.


    You will find two source codebases in this project:

  * **CallObfuscatorPlugin**: The actual plugin, written in C++, that will be compiled and linked to a dll.
      * **CallObfuscator**: Includes the logic to transparently apply call obfucation at compile time.
    * **CallObfuscatorPass**: Initalization and management of the obfuscator pass.
    * **CallObfuscatorPluginRegister**: Plugin registration.


  * **CallObfuscatorHelpers**: A C library that includes all the logic that needs to be executed at runtime.
    * **common**: Common functionality that is used across the project.
    * **pe**: Utilities to manipulate and work with in-memory PEs.
    * **callDispatcher**: Functionality to invoke Windows native functions applying obfuscation.
    * **stackSpoof**: Functionality to apply dynamic stack spoofing in Windows x64 environments.
    * **syscalls**: Utilities to work with Windows x64 syscalls.


* ### How the pass works
    ---
    Knoledge about indirect syscalling, dynamic stack spoofing and common terms like hooks, register, stack... is assumed.
    This is not an in-depth guide, just enough to get you throw the execution flow.

    First, we go through every defined function in the code; if any of them is found in the config file, we store it. Once we find all the functions that will be obfuscated, we create two tables:
    * ```__callobf_dllTable```: This contains all required dlls for obfuscated functions; each dll has an ID, which is its index in the table.
    * ```__callobf_functionTable```: This contains all obfuscated functions and information about which dll contains them, the number of arguments of the function, if it is a syscall, etc.

    At compile time, this tables will be partially initialized, but the only value we need at this moment is the function ID (its index in the function table).

    After building the tables, we find every call to the obfuscated functions; for each of them, replace the call by a call to ```__callobf_callDispatcher```, and pass the id as the first argument, then pass all the function arguments.

    ```__callobf_callDispatcher``` has is defined as ```PVOID __callobf_callDispatcher(DWORD32 index, ...)```. It will get all the info it needs from the function table by using the ID (index) in the first argument.

    A function has its entry partially initialized until it is called; at that moment, ```__callobf_callDispatcher``` will store all the required information to call and obfuscate the function and pass the other arguments to the function being called.

* ### How the dispatching system works
    ---
    The dispatching system starts by initializing the frame table (```__callobf_globalFrameTable```), used to cache posible frames and gadgets that will be used to build the obfuscated stack. The obfuscation method is the same as explained [here](https://klezvirus.github.io/RedTeaming/AV_Evasion/StackSpoofing/), still an outstanding job.

    When ```__callobf_globalFrameTable``` gets invoked, the following happens:
  * Loads the function if needed:
    * Gets the dll from the dll table and loads it if needed.
    * Find the function in the IAT and store the address in the function table.
    * Find if the call is a syscall; if it is, get the ssn and store it in the function table.

  * Build a fake stack using the values stored in the frame table     and store it over the current stack pointer.
  * Update the ciclic value used to pick which values are used for    building the stack.
  * Move arguments to their right place.
  * Change rsp to match the start of the fake stack.
  * If syscall, set ssn in rax.
  * If syscall, set r10 to hold the first argument.
  * Jump to the function or syscall instruction.

## Thanks
To Arash Parsa, aka [waldoirc](https://twitter.com/waldoirc), Athanasios Tserpelis, aka [trickster0](https://twitter.com/trickster012) and Alessandro Magnosi, aka [klezVirus](https://twitter.com/klezVirus) because of [SilentMoonwalk](https://klezvirus.github.io/RedTeaming/AV_Evasion/StackSpoofing/)

---
> ## TODO:
> This includes things that I really dont want to forget, but more stuff could be added here. Not by now
>### Docs/formatting:
>* Document functions
>* Check that the setup instruction works.
>* Somehow improve stackSpoofHelper.x64.asm readability (not today, it works)
>  
>### Opsec:
>* EAF bypass.
>* Seach modules by its hash.
>* Encript strings that cant be hashed.
>
>### General quality:
>* Group all globals, or somehow make it clear in the code where all globals are declared.
>* Put MIN_ADD_RSP_FRAME_SIZE to work.
>* Better error propagation and implement callObfGetLastError()
>* Optionally validate config file entries against local dlls
>* Optionally return load errors through messagebox pop ups, similarly to how ms does with CRTs
>
>### Functionality:
>* Handle invoke instructions and exception stuff (should not happen in C but...)
>* Handle indirect calls
>* Handle function address reads