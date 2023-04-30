---
title: How to easily build a macos application without a mac by using CI/CD tools
slug: macos_builds_ci_cd
date: 2023-04-30T07:43:45-04:00
description: 
draft: false
tags:
    - CI/CD
    - appveyor
    - macos
    - Lua
    - SDL2
---

## Introduction

I've been making a C++ game engine from scratch in the past few months, and one of the first things I worked on after getting the basics working was making sure that the engine compiled and worked on all the platforms that I'm planning on supporting. Building the macos version has been particularly difficult because I don't have a mac, and unlike everything else I'm targeting I can't just cross compile from Linux.

In this blog post I will explain how I solved this problem by making my macos builds using CI/CD tools (specifically [appveyor](https://www.appveyor.com/)), through a toy example of an application that uses [SDL2](https://www.libsdl.org/) and [Lua](https://www.lua.org/).

The complete project that I describe in this blog post [can be found here](https://github.com/apicici/appveyor-macos-build-example).

## The toy application

The test application is all in a single `main.cpp` file, and doesn't really do anything interesting.

```c++
#include "SDL2/SDL.h"
#include "lua.hpp"

int main(int argc, char *argv[])
{
    lua_State *L = luaL_newstate();
    luaL_openlibs(L);
    luaL_dostring(L, "print('Lua is working')");
    lua_close(L);

    if (SDL_Init(SDL_INIT_EVERYTHING) != 0) {
        printf("error initializing SDL: %s\n", SDL_GetError());
    }
    SDL_Window* window = SDL_CreateWindow("Test",
                                          SDL_WINDOWPOS_CENTERED,
                                          SDL_WINDOWPOS_CENTERED,
                                          100, 100, 0); 
    int close = false;
 
    while (!close) {
        SDL_Event event;
 
        while (SDL_PollEvent(&event)) {
            if (event.type == SDL_QUIT) {
                close = true;
            }
        }
        SDL_Delay(100);
    }
 
    SDL_DestroyWindow(window);
    SDL_Quit();
    return 0;
}
```

## CMake project

I like to use [CMake](https://cmake.org/) for my cross-platform projects, so that's how I'm setting up this example. We could just as easily do this directly with a Makefile. The `CMakeLists.txt` file is the following.

```cmake
cmake_minimum_required(VERSION 3.18...3.23)

set(PROJECT_NAME "SDL2_Lua_test")
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_OSX_ARCHITECTURES "arm64;x86_64")
set(CMAKE_OSX_DEPLOYMENT_TARGET 10.11)

project(
    ${PROJECT_NAME}
    VERSION 0.1
    LANGUAGES C CXX
)

add_executable(${PROJECT_NAME} MACOSX_BUNDLE src/main.cpp)

find_package(SDL2 REQUIRED)
find_package(Lua REQUIRED) 

target_include_directories(${PROJECT_NAME} PUBLIC
    ${SDL2_INCLUDE_DIRS}
    ${LUA_INCLUDE_DIR}
)

target_link_libraries(${PROJECT_NAME} PUBLIC SDL2::SDL2 ${LUA_LIBRARY})
```

If you are familiar with CMake the non-macos-specific lines should be easy to parse. For the macos lines:

* `set(CMAKE_OSX_ARCHITECTURES "arm64;x86_64")` is needed to make sure that the application is built as a universal binary that works on both x86_64 and the new arm-based macs.
* `set(CMAKE_OSX_DEPLOYMENT_TARGET 10.11)` sets the minimum version of macos that will be needed to run the application. I'm setting it to `10.11` here because that's the minimum required by the SDL2 version I'm linking to.
* The `MACOSX_BUNDLE` option in `add_executable` tells CMake to make an app bundle out of the executable. *Note that you will probably want to customise the app bundle for your own application when you make builds (for example by adding an icon), but explaining how to do that is out of the scope of this blog post.*

## Building with appveyor

The CMake project described above builds without problems on Linux, and it would build on a mac if you had a mac. But you don't have a mac, or you wouldn't be reading this!

My solution is to set up a git repository for the application, and connect it to appveyor. The following assumes that you already know how to do that.

The configuration of appveyor for this task is quite simple. The content of `appveyor.yml` is
```yml
platform: x64
skip_non_tags: true
clone_depth: 1

environment:
  matrix:
    - job_name: macOS
      appveyor_build_worker_image: macos-bigsur

build_script:
  - sh: bash .appveyor/mac.sh
```

This creates a simple job using a macos worker, with all the commands needed for the build inside the `mac.sh` bash script.

### The bash script

There are 4 things that we need to do in the bash script:
1. Build Lua, as there are no precompiled releases
2. Download SDL2 and add it to the worker
3. Build the application bundle using CMake
4. Create an artifact from the application bundle so that we can download it after it's built

To make things simpler, I will split the `mac.sh` file in 4 parts to go over each of these things separately.

#### Building Lua

```bash
LUA_VERSION=5.4.4
export MACOSX_DEPLOYMENT_TARGET=10.11

mkdir -p "$HOME/Library/Frameworks"

mkdir tmp
cd tmp

# dowload and build universal dylib for Lua, following
# https://blog.spreendigital.de/2015/01/22/how-to-compile-lua-5-3-0-as-a-mac-os-x-dynamic-library/
curl -L https://www.lua.org/ftp/lua-${LUA_VERSION}.tar.gz --output lua-${LUA_VERSION}.tar.gz
tar zxf lua-${LUA_VERSION}.tar.gz
cd lua-${LUA_VERSION}
make macosx MYCFLAGS="-arch x86_64 -arch arm64"
echo 'liblua.dylib: $(CORE_O) $(LIB_O)' >> src/makefile
echo -e '\t$(CC) -dynamiclib -o $@ $^ $(LIBS) -arch x86_64 -arch arm64 -install_name @rpath/$@' >> src/makefile
make -C src liblua.dylib
cp src/*.{h,hpp,dylib} "$HOME/Library/Frameworks/"
cd ..
```
This part is probably the most complicated one, and it's due to the fact that the Lua Makefile is not set up to build a shared library, so we need to add an extra rule to do that for us.

Setting `MACOSX_DEPLOYMENT_TARGET` is needed to make sure that the compiler sets the appropriate minimum version (otherwise the worker version is used by default)

Once `liblua.dylib`, we copy it with the appropriate header files to `$HOME/Library/Frameworks/`, where CMake will be able to find them.

#### Getting SDL2

This one is quite easy, since there are prebuilt version already set up as macos frameworks.

```sh
curl -L https://github.com/libsdl-org/SDL/releases/download/release-2.26.5/SDL2-2.26.5.dmg --output SDL2-2.26.5.dmg
hdiutil attach SDL2-2.26.5.dmg
cp -a /Volumes/SDL2/SDL2.framework "$HOME/Library/Frameworks/SDL2.framework"
```

#### Building the app bundle

```sh
cd ..
mkdir build
cd build
cmake ..
make

# set rpath to bundle Lua and SDL2 with the executable inside the app bundle
install_name_tool -add_rpath @executable_path SDL2_Lua_test.app/Contents/MacOS/SDL2_Lua_test
cp -a "$HOME/Library/Frameworks/SDL2.framework" SDL2_Lua_test.app/Contents/MacOS/
cp -a "$HOME/Library/Frameworks/"*.dylib SDL2_Lua_test.app/Contents/MacOS/
```
The first half here is just standard CMake building. Things get more complicated in the second half because we need to make sure that the dependencies (SDL2 and Lua) are bundled inside the `.app` and are loaded from there by our executable.
To achieve this, we do the following:
* Copy `SDL2.framework` and `liblua.dylib` inside the app bundle, in the same directory as the executable (i.e., `SDL2_Lua_test.app/Contents/MacOS/`)
* Set the `rpath` of the executable so that it loads dynamic libraries from the folder where it is located (`@executable_path`)

I suggest reading [here](https://blog.krzyzanowskim.com/2018/12/05/rpath-what/) for more information on the rpath steps.

#### Creating the artifact

Finally, we do this:
```sh
zip -r SDL2_Lua_test.zip SDL2_Lua_test.app
appveyor PushArtifact SDL2_Lua_test.zip
```
This allows us to download the app bundle we created from the 'Artifacts' of the current build on appveyor.

That's it, it's all done!

## Some tips
Some additional tips if you are interested in trying this approach:
* Testing the build by changing the bash file, pushing to git, and restarting the appveyor build is time consuming and extremely frustrating. I strongly recommend 	[setting up appveyor for ssh access](https://www.appveyor.com/docs/how-to/ssh-to-build-worker/) so you can test from the inside in real time!
* You can also set up appveyor to deploy the artifact somewhere, for example as a github release.
* This approach works also for Windows and Linux builds!
