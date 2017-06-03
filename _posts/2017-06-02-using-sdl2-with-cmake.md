---
layout: post
title:  "Using SDL2 with CMake"
date:   2017-06-02 16:42:00 +0200
feature_image: "https://unsplash.it/1200/400?image=990"
categories: Development
tags: [sdl2, cmake]
---
SDL2 is the newest version of the Simple Directmedia Layer API. It can be used
together with CMake to build a cross platform multimedia application. In this
blog post I will describe the necessary steps to use SDL2 with CMake on both
Linux (Ubuntu 17.04) and  Windows.

<!-- more -->

# Linux

Setting up SDL2 with CMake under Ubuntu Linux is pretty easy. All you need to do
is install the required dependencies first.

{% highlight bash %}
sudo apt install cmake libsdl2-dev g++
{% endhighlight %}

And then create a CMakeLists.txt file for your project.

{% highlight cmake %}
cmake_minimum_required(VERSION 3.7)

project(SDL2Test)

find_package(SDL2 REQUIRED) include_directories(SDL2Test ${SDL2_INCLUDE_DIRS})

add_executable(SDL2Test Main.cpp) target_link_libraries(SDL2Test ${SDL2_LIBRARIES})
{% endhighlight %}

Now you can use `cmake` to generate your Makefiles and build your project. To
include SDL2 headers you just use `#include "SDL.h"`. The correct include paths
have been set up by cmake.

# Windows

For Windows you have to download the development package
`SDL2-devel-2.0.5-VC.zip` and extract it to some location on your hard disk.

You can create Visual Studio project files with the CMake GUI under windows but
when you hit configure it will fail because it will not find the SDL2 Library.

In the configuration window you will see a SDL2_DIR variable. You will have to
point that to the location where you extracted the SDL2 development package.

Before you can reconfigure you also have to create a file `sdl2-config.cmake`
where you extracted the development libraries and put the following content:

{% highlight cmake %}
set(SDL2_INCLUDE_DIRS "${CMAKE_CURRENT_LIST_DIR}/include")

# Support both 32 and 64 bit builds
if (${CMAKE_SIZEOF_VOID_P} MATCHES 8)
  set(SDL2_LIBRARIES "${CMAKE_CURRENT_LIST_DIR}/lib/x64/SDL2.lib;${CMAKE_CURRENT_LIST_DIR}/lib/x64/SDL2main.lib")
else ()
  set(SDL2_LIBRARIES "${CMAKE_CURRENT_LIST_DIR}/lib/x86/SDL2.lib;${CMAKE_CURRENT_LIST_DIR}/lib/x86/SDL2main.lib")
endif ()

string(STRIP "${SDL2_LIBRARIES}" SDL2_LIBRARIES)
{% endhighlight %}

After that you now should be able to reconfigure successfully and generate the
Visual Studio project files.

# Test App

You can create a test application to verify that everything works. You could for instance use the following code for that which renders a black window.

{% highlight cpp %}
#include "SDL.h"


int main(int argc, char *argv[])
{
  SDL_Init(SDL_INIT_VIDEO);

  SDL_Window *window = SDL_CreateWindow(
    "SDL2Test",
    SDL_WINDOWPOS_UNDEFINED,
    SDL_WINDOWPOS_UNDEFINED,
    640,
    480,
    0
  );

  SDL_Renderer *renderer = SDL_CreateRenderer(window, -1, SDL_RENDERER_SOFTWARE);
  SDL_SetRenderDrawColor(renderer, 0, 0, 0, SDL_ALPHA_OPAQUE);
  SDL_RenderClear(renderer);
  SDL_RenderPresent(renderer);

  SDL_Delay(3000);

  SDL_DestroyWindow(window);
  SDL_Quit();

  return 0;
}
{% endhighlight %}