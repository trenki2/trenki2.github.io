---
layout: post
title:  "Using SDL2_image with CMake"
date:   2017-07-04 10:13:00 +0200
last_modified_at: 2019-01-13 18:00:00 +0200
feature_image: "https://unsplash.it/1200/400?image=990"
category: Development
tags: [sdl2, sdl2_image, cmake]
---

SDL2_image is an add-on library for SDL2, the Simple DirectMedia Layer and can
be used to load images from various formats. In this post I describe how to get
it working with CMake under Windows and Linux.

<!-- more -->

You might also want to check out my other post [Using SDL2 with CMake]({%
post_url 2017-06-02-using-sdl2-with-cmake %}).

## CMake Module

To get SDL2_image working with CMake you need a `FindSDL2_image.cmake` CMake
module. Unfortunately CMake does not have one for SDL2_image but it has one for
the older SDL_image which can be adapted to work with SDL2_image.

This CMake module needs to be places somewhere inside your project source tree
for instance into a `cmake` subdirectory. You then need to also set the CMake
module path.

```cmake
set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_SOURCE_DIR}/cmake/")
```

When this is done you can find the `SDL2_image` package with the `find_package`
command. You might also have to set the `SDL2_DIR` cmake variable so that the
SDL directory is found.

```cmake
find_package(SDL2 REQUIRED)
find_package(SDL2_image REQUIRED)
include_directories(${SDL2_INCLUDE_DIRS} ${SDL2_IMAGE_INCLUDE_DIRS})
target_link_libraries(Exe ${SDL2_LIBRARIES} ${SDL2_IMAGE_LIBRARIES})
```

The source of the `FindSDL2_image.cmake` file that I am using is this one:

```cmake
# Distributed under the OSI-approved BSD 3-Clause License.  See accompanying
# file Copyright.txt or https://cmake.org/licensing for details.

#.rst:
# FindSDL2_image
# -------------
#
# Locate SDL2_image library
#
# This module defines:
#
# ::
#
#   SDL2_IMAGE_LIBRARIES, the name of the library to link against
#   SDL2_IMAGE_INCLUDE_DIRS, where to find the headers
#   SDL2_IMAGE_FOUND, if false, do not try to link against
#   SDL2_IMAGE_VERSION_STRING - human-readable string containing the
#                              version of SDL2_image
#
#
#
# For backward compatibility the following variables are also set:
#
# ::
#
#   SDL2IMAGE_LIBRARY (same value as SDL2_IMAGE_LIBRARIES)
#   SDL2IMAGE_INCLUDE_DIR (same value as SDL2_IMAGE_INCLUDE_DIRS)
#   SDL2IMAGE_FOUND (same value as SDL2_IMAGE_FOUND)
#
#
#
# $SDLDIR is an environment variable that would correspond to the
# ./configure --prefix=$SDLDIR used in building SDL.
#
# Created by Eric Wing.  This was influenced by the FindSDL.cmake
# module, but with modifications to recognize OS X frameworks and
# additional Unix paths (FreeBSD, etc).

if(NOT SDL2_IMAGE_INCLUDE_DIR AND SDL2IMAGE_INCLUDE_DIR)
  set(SDL2_IMAGE_INCLUDE_DIR ${SDL2IMAGE_INCLUDE_DIR} CACHE PATH "directory cache
entry initialized from old variable name")
endif()
find_path(SDL2_IMAGE_INCLUDE_DIR SDL_image.h
  HINTS
    ENV SDL2IMAGEDIR
    ENV SDL2DIR
    ${SDL2_DIR}
  PATH_SUFFIXES SDL2
                # path suffixes to search inside ENV{SDL2DIR}
                include/SDL2 include
)

if(CMAKE_SIZEOF_VOID_P EQUAL 8)
  set(VC_LIB_PATH_SUFFIX lib/x64)
else()
  set(VC_LIB_PATH_SUFFIX lib/x86)
endif()

if(NOT SDL2_IMAGE_LIBRARY AND SDL2IMAGE_LIBRARY)
  set(SDL2_IMAGE_LIBRARY ${SDL2IMAGE_LIBRARY} CACHE FILEPATH "file cache entry
initialized from old variable name")
endif()
find_library(SDL2_IMAGE_LIBRARY
  NAMES SDL2_image
  HINTS
    ENV SDL2IMAGEDIR
    ENV SDL2DIR
    ${SDL2_DIR}
  PATH_SUFFIXES lib ${VC_LIB_PATH_SUFFIX}
)

if(SDL2_IMAGE_INCLUDE_DIR AND EXISTS "${SDL2_IMAGE_INCLUDE_DIR}/SDL2_image.h")
  file(STRINGS "${SDL2_IMAGE_INCLUDE_DIR}/SDL2_image.h" SDL2_IMAGE_VERSION_MAJOR_LINE REGEX "^#define[ \t]+SDL2_IMAGE_MAJOR_VERSION[ \t]+[0-9]+$")
  file(STRINGS "${SDL2_IMAGE_INCLUDE_DIR}/SDL2_image.h" SDL2_IMAGE_VERSION_MINOR_LINE REGEX "^#define[ \t]+SDL2_IMAGE_MINOR_VERSION[ \t]+[0-9]+$")
  file(STRINGS "${SDL2_IMAGE_INCLUDE_DIR}/SDL2_image.h" SDL2_IMAGE_VERSION_PATCH_LINE REGEX "^#define[ \t]+SDL2_IMAGE_PATCHLEVEL[ \t]+[0-9]+$")
  string(REGEX REPLACE "^#define[ \t]+SDL2_IMAGE_MAJOR_VERSION[ \t]+([0-9]+)$" "\\1" SDL2_IMAGE_VERSION_MAJOR "${SDL2_IMAGE_VERSION_MAJOR_LINE}")
  string(REGEX REPLACE "^#define[ \t]+SDL2_IMAGE_MINOR_VERSION[ \t]+([0-9]+)$" "\\1" SDL2_IMAGE_VERSION_MINOR "${SDL2_IMAGE_VERSION_MINOR_LINE}")
  string(REGEX REPLACE "^#define[ \t]+SDL2_IMAGE_PATCHLEVEL[ \t]+([0-9]+)$" "\\1" SDL2_IMAGE_VERSION_PATCH "${SDL2_IMAGE_VERSION_PATCH_LINE}")
  set(SDL2_IMAGE_VERSION_STRING ${SDL2_IMAGE_VERSION_MAJOR}.${SDL2_IMAGE_VERSION_MINOR}.${SDL2_IMAGE_VERSION_PATCH})
  unset(SDL2_IMAGE_VERSION_MAJOR_LINE)
  unset(SDL2_IMAGE_VERSION_MINOR_LINE)
  unset(SDL2_IMAGE_VERSION_PATCH_LINE)
  unset(SDL2_IMAGE_VERSION_MAJOR)
  unset(SDL2_IMAGE_VERSION_MINOR)
  unset(SDL2_IMAGE_VERSION_PATCH)
endif()

set(SDL2_IMAGE_LIBRARIES ${SDL2_IMAGE_LIBRARY})
set(SDL2_IMAGE_INCLUDE_DIRS ${SDL2_IMAGE_INCLUDE_DIR})

include(FindPackageHandleStandardArgs)

FIND_PACKAGE_HANDLE_STANDARD_ARGS(SDL2_image
                                  REQUIRED_VARS SDL2_IMAGE_LIBRARIES SDL2_IMAGE_INCLUDE_DIRS
                                  VERSION_VAR SDL2_IMAGE_VERSION_STRING)

# for backward compatibility
set(SDL2IMAGE_LIBRARY ${SDL2_IMAGE_LIBRARIES})
set(SDL2IMAGE_INCLUDE_DIR ${SDL2_IMAGE_INCLUDE_DIRS})
set(SDL2IMAGE_FOUND ${SDL2_IMAGE_FOUND})

mark_as_advanced(SDL2_IMAGE_LIBRARY SDL2_IMAGE_INCLUDE_DIR)
```

There is also a complete project in my github repository [here](https://github.com/trenki2/SDL2Test).

