---
layout: post
title:  "Developing a Software Renderer"
date:   2017-06-05 08:22:00 +0200
feature_image: "https://unsplash.it/1200/400?image=41"
categories: [Development, Software Rendering]
tags: [rasterization, rendering, sdl2, c++]
excerpt: |
  This article is about graphics programming. I implemented my own compact software
  renderer/rasterizer with some nice features like pixel and vertex shaders in C++
  and in this article I describe how I did it.
---

Today software rendering has mostly been replaced by GPUs but there are still
places where it can be useful.

One example is software based occlusion culling ([Software Occlusion
Culling](https://software.intel.com/en-us/articles/software-occlusion-culling)
and [Masked Occlusion
Culling](https://github.com/GameTechDev/MaskedOcclusionCulling)) where a
software renderer is used to create a hierarchical z-buffer which is in turn
used to test object visibility and prevent invisible stuff from being sent to
the GPU.

I implemented my own compact software renderer/rasterizer with some nice
features like pixel and vertex shaders in C++ and in this article I describe how
I did it.

<!-- more -->

## Setting Up The Environment

We need to create a window where we can render our stuff into. For this we will
use SDL2. It works under Windows and Linux. My blog post [Using SDL2 with
CMake]({% post_url 2017-06-02-using-sdl2-with-cmake %}) describes all the
necessary steps required to setup SDL2 with CMake.

Once you have set this up you can remove the code which uses the `SDL_Renderer`
and use the following to render directly to the screen without a renderer:

```cpp
SDL_Surface *screen = SDL_GetWindowSurface(window);
SDL_FillRect(screen, 0, 0);
SDL_UpdateWindowSurface(window);
```

## Drawing Pixels

Ultimately we want to rasterize a triangle. For this we need to be able to fill
every pixel inside the triangle with some color. We need a `putpixel` function
to accomplish this.

An implementation of such a function can be found
[here](http://sdl.beuc.net/sdl.wiki/Pixel_Access) or
[here](https://www.libsdl.org/release/SDL-1.2.15/docs/html/guidevideo.html).

You can then draw a lot of pixels like this:

```cpp
for (int i = 0; i < 10000; i++)
{
  int x = random() % 640;
  int y = random() % 480;
  int r = random() % 255;
  int g = random() % 255;
  int b = random() % 255;

  putpixel(screen, x, y, SDL_MapRGB(screen->format, r, g, b));
}
```

## Rasterizing a Triangle

There are a lot of resources about triangle rasterization available online but I
feel like a lot of that information is not very good.

Fortunately I was able to find two valuable resources about triangle
rasterization that helped greatly.

* [Triangle Rasterization](http://www.cs.unc.edu/~blloyd/comp770/Lecture08.pdf)
* [Accelerated Half-Space Triangle Rasterization](https://www.researchgate.net/publication/286441992_Accelerated_Half-Space_Triangle_Rasterization)

The interested reader should read both of those resources to understand the
inner workings of the rasterizer.

### Simple Filling

I decided to implement a rasterizer based on edge equations. We can encapsulate
the edge related operation in an `EdgeEquation` class.


```cpp
struct EdgeEquation {
  float a;
  float b;
  float c;
  bool tie; 

  EdgeEquation(const Vertex &v0, const Vertex &v1)
  {
    a = v0.y - v1.y;
    b = v1.x - v0.x;
    c = - (a * (v0.x + v1.x) + b * (v0.y + v1.y)) / 2;
    tie = a != 0 ? a > 0 : b > 0;
  }

  /// Evaluate the edge equation for the given point.

  float evaluate(float x, float y)
  {
    return a * x + b * y + c;
  }

  /// Test if the given point is inside the edge.

  bool test(float x, float y)
  {
    return test(evaluate(x, y));
  }

  /// Test for a given evaluated value.
  
  bool test(float v)
  {
    return (v > 0 || v == 0 && tie);
  }
};
```

Then we can go on and rasterize the triangle. We compute the bounding box of the
triangle, restrict it to the scissor rectangle, cull backfacing triangles and
then fill all the pixels inside the triangle while obeying the fill rule with
the tie-breaker as described in the references.

```cpp
void drawTriangle(const Vertex& v0, const Vertex &v1, const Vertex &v2)
{
  // Compute triangle bounding box.

  int minX = std::min(std::min(v0.x, v1.x), v2.x);
  int maxX = std::max(std::max(v0.x, v1.x), v2.x);
  int minY = std::min(std::min(v0.y, v1.y), v2.y);
  int maxY = std::max(std::max(v0.y, v1.y), v2.y);

  // Clip to scissor rect.
  
  minX = std::max(minX, m_minX);
  maxX = std::min(maxX, m_maxX);
  minY = std::max(minY, m_minY);
  maxY = std::min(maxY, m_maxY);

  // Compute edge equations.

  EdgeEquation e0(v0, v1);
  EdgeEquation e1(v1, v2);
  EdgeEquation e2(v2, v0);

  float area = 2 * (e0.c + e1.c + e2.c);

  // Check if triangle is backfacing.

  if (area < 0)
    return;

  // Add 0.5 to sample at pixel centers.

  for (float x = minX + 0.5f, xm = maxX + 0.5f; x <= xm; x += 1.0f)
  for (float y = minY + 0.5f, ym = maxY + 0.5f; y <= ym; y += 1.0f)
  {
    if (e0.test(x, y) && e1.test(x, y) && e2.test(x, y))
      putpixel(m_surface, x, y, 0xffffffff);
  }
}

```

![Screenshot]({{ site.url }}/assets/images/software-rendering/triangle1.png){: .align-center}

### Adding Color

### Block Based

The simple implementation works, but performance is not great. It can be
improved by a block based approach that allows us to discard blocks outside the
triangle faster and skip some test when the block is completely inside the
triangle.

The block based approach gives better performance and can also be parallelized