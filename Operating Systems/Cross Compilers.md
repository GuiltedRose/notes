## What is a Cross Compiler?
This is a compiler for a programming language like C/C++, Rust, etc. that utilizes all the features minus the ones that our current operating system uses. For example: if we are using linux and compile our kernel with gcc it could bring in dependencies from our native operating system. We want to avoid this so that nothing important breaks this is usually referred to as a =="freestanding binary"==. 
## Setting Up a Cross Compiler:
In order to do this we can go to [OS Dev](https://wiki.osdev.org/GCC_Cross-Compiler). Once there, we can download all the tools it tells us we need to build our own cross compiler. The reason we are doing this is because our current compiler is already linked to our native operating system (it should be linux for a much easier experience). After that we need the source files for `build-essentials` & `gcc` both are developed by the GNU foundation. Once downloaded we will extract them to our home folder (because we are going to use this OS eventually it shouldn't really matter too much). They are also super easy to clean up after we're done with them.
![dependency-list.png](https://github.com/GuiltedRose/notes/blob/main/pictures/dependency-list.png?raw=true)
![arch-package-list.png](https://github.com/GuiltedRose/notes/blob/main/pictures/arch-package-list.png?raw=true)
Above are the list of dependencies we need for the gcc cross compiler, luckily on arch most if not all of these are installed by default.

```sh
export PREFIX="$HOME/opt/cross"
export TARGET=i686-elf
export PATH="$PREFIX/bin:$PATH"
```

This is one the most important step in the cross compilation process, as this lets us specify exactly which architecture we are building gcc for, and will help us create our own C/C++ kernel much easier.

I also wrote these helper scripts: `cross-compile-setup.sh` & `cleanup.sh`. These files will allow us to always download the specific version of gcc & binutils that our project needs.

We can then follow the process as seen on the OS Dev article linked above. I also went ahead and created a f a build script.
```sh
#!/bin/bash

export PREFIX="$HOME/opt/cross"
export TARGET=i686-elf
export PATH="$PREFIX/bin:$PATH"

make all
```
This will ensure we can always compile our code with our cross compiler.

Eventually we will be making our own linker and compiler and will no longer need GNU tools to get the job done, but until then, we will keep pushing.