---
layout: post
title:  "Developing a Software Renderer Part 3"
date:   2017-06-15 9:56:00 +0200
#last_modified_at: 2017-06-10 10:10:00 +0200
feature_image: "https://unsplash.it/1200/400?image=41"
category: Software Rendering
tags: [rasterization, rendering, c++]
---
In this post I describe how to add pixel shader capabilities to the software
rasterizer and how to optimize it even further for example using OpenMP to
parallelize the rasterization.

<!-- more -->

## Pixel Shader Support

Currently the per pixel operations are coded directly inside the innermost loop.
We want to have the ability to do arbitrary work per pixel and use the
rasterizer as a library.

One simple way to do this would be to define an interface `IPixelShader` with a
virtual  method `drawPixel` and pass this to the rasterizer. I already tried
this out and found out that the virtual function call is way to expensive and
slows everything down too  much. The second best option would be a function
pointer but I assumed this will be slow too.

The best thing would be if the compiler could inline our per pixel code, so that
there is no function call at all. It turns out there is such a possibility with
C++ templates.

We can parameterize the whole triangle rasterization method with our pixel
shader class using a c++ template and call a static method in the pixel shader
class to draw our pixels. This way the compiler can inline the pixel drawing
method inside the inner loop and this should give the best performance.

The pixel shader class will then look like this:

```cpp
class PixelShader : public PixelShaderBase<PixelShader> {
public:
  static const bool InterpolateZ = false;
  static const bool InterpolateW = false;
  static const int VarCount = 3;

  static SDL_Surface* surface;

  static void drawPixel(const PixelData &p)
  {
    int rint = (int)(p.var[0] * 255);
    int gint = (int)(p.var[1] * 255);
    int bint = (int)(p.var[2] * 255);

    Uint32 color = rint << 16 | gint << 8 | bint;

    Uint32 *buffer = (Uint32*)((Uint8 *)surface->pixels + (int)p.y * surface->pitch + (int)p.x * 4);
    *buffer = color;
  }
};
```

The pixel shader can tell the rasterizer if the z and w component are supposed
to be interpolated and it can also tell the rasterizer how many per vertex
attributes to interpolate (`VarCount`). Our updated `Vertex` structure now
supports arbitrary variables not just RGB color.

```cpp
struct Vertex {
  float x;
  float y;
  float z;
  float w;

  float var[MaxVar];
};
```

To use the pixel shader we can use the following code:

```cpp
rasterizer.setPixelShader<PixelShader>();
rasterizer.drawTriangle(v0, v1, v2);
```

The trick here is that the `setPixelShader<PixelShader>()` method gets a member
function pointer to the actual `drawTriangleTemplate<PixelShader>` function and
stores it in a variable so that the `drawTriangle` call does not require the
template any longer and will just forward the call to the stored member
function.

## Parallelization Using OpenMP

With OpenMP we can parallelize our software rasterizer in a simple way.
We can use some `#pragma` statements to do this.

To use OpenMP with CMake we can use the following code in the `CMakeLists.txt`
file:

```cmake
find_package(OpenMP)
if (OPENMP_FOUND)
  set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${OpenMP_C_FLAGS}")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${OpenMP_CXX_FLAGS}")
endif ()
```

In the rasterizer we can parallelize the for loop which iterates over all the
blocks that need to be drawn. We could prepend the following pragma and be done:

```cpp
#pragma omp parallel for collapse(2)
```

Unfortunately `collapse` only works on Linux since there we have OpenMP 3.0.
Under Windows with Visual Studio we need a workaround. We need to restructure
the loop so that we do not have any nesting.

```cpp
int stepsX = (maxX - minX) / BlockSize + 1;
int stepsY = (maxY - minY) / BlockSize + 1;

#pragma omp parallel for

for (int i = 0; i < stepsX * stepsY; ++i)
{
  int sx = i % stepsX;
  int sy = i / stepsX;

  int x = minX + sx * BlockSize;
  int y = minY + sy * BlockSize;

  // Add 0.5 to sample at pixel centers.

  float xf = x + 0.5f;
  float yf = y + 0.5f;

  [...]
```

When we now run the rasterizer it is indeed faster but on my machine I only
achieved a speedup of 2x.

## Scanline Triangle Rasterization

After experimenting a bit I was not satisfied with the performance of the
rasterizer and I tried some alternative approaches. Instead of doing the block
based rasterization I followed a scanline based rasterization approach as
desribed in [Software Rasterization Algorithms for Filling
Triangles](http://www.sunshine2k.de/coding/java/TriangleRasterization/TriangleRasterization.html).

This approach uses the fact that there are two special cases, a bottom flat and
top flat triangle, which are easy to rasterize. It splits each triangle into two
such easy triangles and rasterizes them individually.

```cpp
template <class PixelShader>
void drawTriangleSpanTemplate(const Vertex& v0, const Vertex &v1, const Vertex &v2) const
{
  // Compute triangle equations.

  TriangleEquations eqn(v0, v1, v2, PixelShader::VarCount);

  // Check if triangle is backfacing.

  if (eqn.area2 <= 0)
    return;

  const Vertex *t = &v0;
  const Vertex *m = &v1;
  const Vertex *b = &v2;

  // Sort vertices from top to bottom.

  if (t->y > m->y) std::swap(t, m);
  if (m->y > b->y) std::swap(m, b);
  if (t->y > m->y) std::swap(t, m);

  float dy = (b->y - t->y);
  float iy = (m->y - t->y);

  // Trivial case top-flat triangle

  if (m->y == t->y)
  {
    const Vertex *l = m, *r = t;
    if (l->x > r->x) std::swap(l, r);
    drawTopFlatTriangle<PixelShader>(eqn, *l, *r, *b);
  }
  // Trivial case bottom-flat triangle

  else if (m->y == b->y)
  {
    const Vertex *l = m, *r = b;
    if (l->x > r->x) std::swap(l, r);
    drawBottomFlatTriangle<PixelShader>(eqn, *t, *l, *r);
  }
  else
  {
    // General case - split the triangle

    Vertex v4;
    v4.y = m->y;
    v4.x = t->x + ((b->x - t->x) / dy) * iy;
    if (PixelShader::InterpolateZ) v4.z = t->z + ((b->z - t->z) / dy) * iy;
    if (PixelShader::InterpolateW) v4.w = t->w + ((b->w - t->w) / dy) * iy;
    for (int i = 0; i < PixelShader::VarCount; ++i)
      v4.var[i] = t->var[i] + ((b->var[i] - t->var[i]) / dy) * iy;

    const Vertex *l = m, *r = &v4;
    if (l->x > r->x) std::swap(l, r);

    drawBottomFlatTriangle<PixelShader>(eqn, *t, *l, *r);
    drawTopFlatTriangle<PixelShader>(eqn, *l, *r, *b);
  }
}
```

The triangle filling is shown in the code block below. It can be parallelized
with OpenMP which gives better performance.

```cpp
template <class PixelShader>
void drawBottomFlatTriangle(const TriangleEquations &eqn, const Vertex& v0, const Vertex &v1, const Vertex &v2) const
{
  float invslope1 = (v1.x - v0.x) / (v1.y - v0.y);
  float invslope2 = (v2.x - v0.x) / (v2.y - v0.y);

  #pragma omp parallel for

  for (int scanlineY = int(v0.y + 0.5f); scanlineY < int(v1.y + 0.5f); scanlineY++)
  {
    float dy = (scanlineY - v0.y) + 0.5f;
    float curx1 = v0.x + invslope1 * dy + 0.5f;
    float curx2 = v0.x + invslope2 * dy + 0.5f;

    // Clip to scissor rect

    int xl = std::max(m_minX, (int)curx1);
    int xr = std::min(m_maxX, (int)curx2);

    PixelShader::drawSpan(eqn, xl, scanlineY, xr);
  }
}

template <class PixelShader>
void drawTopFlatTriangle(const TriangleEquations &eqn, const Vertex& v0, const Vertex &v1, const Vertex &v2) const
{
  float invslope1 = (v2.x - v0.x) / (v2.y - v0.y);
  float invslope2 = (v2.x - v1.x) / (v2.y - v1.y);

  #pragma omp parallel for

  for (int scanlineY = int(v2.y - 0.5f); scanlineY > int(v0.y - 0.5f); scanlineY--)
  {
    float dy = (scanlineY - v2.y) + 0.5f;
    float curx1 = v2.x + invslope1 * dy + 0.5f;
    float curx2 = v2.x + invslope2 * dy + 0.5f;

    // Clip to scissor rect

    int xl = std::max(m_minX, (int)curx1);
    int xr = std::min(m_maxX, (int)curx2);

    PixelShader::drawSpan(eqn, xl, scanlineY, xr);
  }
}
```

With the scanline based approach I get between 10-15% better performance when
filling random triangles. Still that does not mean, that the scanline based
approach is better in all the cases.

With the block based approach it would be relatively easy to support some form of
a hierarchical z-buffer and cull whole blocks much faster if there is overdraw.
This is not so simple with the scanline based approach.

One would have to compare the approaches in this specific situation.

## Fixed Point Calculations

I tried out if replacing the floating point calculations in the innermost loops
with fixed point calculations would give better performance. This way the pixel
shader would not get floating point values for its attributes any more but fixed
point values.

I implemented a class `fixed<Bits>` which I used to replace most floating point
calculations that calculated the pixel data but kept the floating point
calculations, which are done per triangle, so the rasterizer still takes
floating point values for the vertex data as input.

I was able to achieve a considerable performance gain of 10-20% with that for
filling random triangles with rgb color. The problem was that there were some
color artifacts, so the `PixelShader` had to be modified to fix the artifacts. I
needed to check for value overflow in the `drawSpan` method of the PixelShader
and depending on the outcome of the test route the processing to a specialized
`drawPixel` method. When I did the overflow test per pixel all the performance
gain was lost.

## Conclusion

The added pixel shader support makes the software renderer very modular and
flexible and with the template based approach we get the best performance out of
it.

Using OpenMP for parallelization we can improve the performance. But also using
a scanline based approach and fixed point math makes quite a big difference.

In the next parts I will develop the vertex processing stage with a vertex cache 
and vertex shader support so that some more complex 3D models can be rendered.

Continue on [Part 4]({% post_url 2017-07-03-developing-a-software-renderer-part4
%})