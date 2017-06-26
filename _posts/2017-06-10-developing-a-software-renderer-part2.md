---
layout: post
title:  "Developing a Software Renderer Part 2"
date:   2017-06-10 10:09:00 +0200
last_modified_at: 2017-06-15 09:56:00 +0200
feature_image: "https://unsplash.it/1200/400?image=41"
category: Software Rendering
tags: [rasterization, rendering, c++]
---
In the second part of this series, I will describe how to optimize and improve
the software rasterizer that we developed in the first part.

<!-- more -->

We will use a block based approach to rasterizing the triangles to be able to
discard regions outside the triangle faster. We will also optimize the inner
loop so that fever computations are required and refactor our code base to make
things simple.

## Refactoring

To make it easy to optimize our algorithm from [Part 1]({% post_url
2017-06-06-developing-a-software-renderer-part1 %}) we will do some refactoring
first.

We add some utility methods to our `EdgeEquation` and `ParameterEquation`
classes to be able to step some given the edge and parameter value `v` along the
x and y axis.

```cpp
struct EdgeEquation {
  [...]

  /// Step the equation value v to the x direction.

  float stepX(float v) const
  {
    return v + a;
  }

  /// Step the equation value v to the x direction.

  float stepX(float v, float stepSize) const
  {
    return v + a * stepSize;
  }

  /// Step the equation value v to the y direction.

  float stepY(float v) const
  {
    return v + b;
  }

  /// Step the equation value vto the y direction.

  float stepY(float v, float stepSize) const
  {
    return v + b * stepSize;
  }
};
```

For the `ParameterEquation` class the methods look identical.

We encapsulate the edge and parameter equations in a `TriangleEquations` class.
This allows us to make the `drawTriangle` method shorter and pass all the
triangle equations to other method. We will need this for the `rasterizeBlock`
method and also for some edge and parameter stepping methods.

```cpp
struct TriangleEquations {
  float area;

  EdgeEquation e0;
  EdgeEquation e1;
  EdgeEquation e2;

  ParameterEquation r;
  ParameterEquation g;
  ParameterEquation b;

  TriangleEquations(const Vertex &v0, const Vertex &v1, const Vertex &v2)
  {
    e0.init(v0, v1);
    e1.init(v1, v2);
    e2.init(v2, v0);

    area = 0.5f * (e0.c + e1.c + e2.c);

    // Cull backfacing triangles.

    if (area < 0)
      return;

    r.init(v0.r, v1.r, v2.r, e0, e1, e2, area);
    g.init(v0.g, v1.g, v2.g, e0, e1, e2, area);
    b.init(v0.b, v1.b, v2.b, e0, e1, e2, area);
  }
};
```

We also declare a `PixelData` class which encapsulates the computed per pixel
data. This class will later be passed to a pixel shader once we have that
framework implemented. We add a `stepX` and `stepY` utility methods that allows
us to step the pixel data to the x and y direction.

Using the step  we can compute the pixel data for the corner of a block that
uses the expensive `evaluate` method and then compute the remaining pixels of
the block by simply stepping the initial value and avoiding to have to call
`evaluate` for every pixel.

```cpp
struct PixelData {
  float r;
  float g;
  float b;

  /// Initialize pixel data for the given pixel coordinates.

  void init(const TriangleEquations &eqn, float x, float y)
  {
    r = eqn.r.evaluate(x, y);
    g = eqn.g.evaluate(x, y);
    b = eqn.b.evaluate(x, y);
  }

  /// Step all the pixel data in the x direction.

  void stepX(const TriangleEquations &eqn)
  {
    r = eqn.r.stepX(r);
    g = eqn.g.stepX(g);
    b = eqn.b.stepX(b);
  }

  /// Step all the pixel data in the y direction.

  void stepY(const TriangleEquations &eqn)
  {
    r = eqn.r.stepY(r);
    g = eqn.g.stepY(g);
    b = eqn.b.stepY(b);
  }
};
```

And finally we implement a `EdgeData` class which encapsulates the computed edge
values for the triangle and also allows for easy stepping along the x and y
direction.

```cpp
struct EdgeData {
  float ev0;
  float ev1;
  float ev2;

  /// Initialize the edge data values.

  void init(const TriangleEquations &eqn, float x, float y)
  {
    ev0 = eqn.e0.evaluate(x, y);
    ev1 = eqn.e1.evaluate(x, y);
    ev2 = eqn.e2.evaluate(x, y);
  }

  /// Step the edge values in the x direction.

  void stepX(const TriangleEquations &eqn)
  {
    ev0 = eqn.e0.stepX(ev0);
    ev1 = eqn.e1.stepX(ev1);
    ev2 = eqn.e2.stepX(ev2);
  }

  /// Step the edge values in the x direction.

  void stepX(const TriangleEquations &eqn, float stepSize)
  {
    ev0 = eqn.e0.stepX(ev0, stepSize);
    ev1 = eqn.e1.stepX(ev1, stepSize);
    ev2 = eqn.e2.stepX(ev2, stepSize);
  }

  /// Step the edge values in the y direction.

  void stepY(const TriangleEquations &eqn)
  {
    ev0 = eqn.e0.stepY(ev0);
    ev1 = eqn.e1.stepY(ev1);
    ev2 = eqn.e2.stepY(ev2);
  }

  /// Step the edge values in the y direction.

  void stepY(const TriangleEquations &eqn, float stepSize)
  {
    ev0 = eqn.e0.stepY(ev0, stepSize);
    ev1 = eqn.e1.stepY(ev1, stepSize);
    ev2 = eqn.e2.stepY(ev2, stepSize);
  }

  /// Test for triangle containment.

  bool test(const TriangleEquations &eqn)
  {
    return eqn.e0.test(ev0) && eqn.e1.test(ev1) && eqn.e2.test(ev2);
  }
};
```

## Block Based Rasterization

With all of these helper classes block based triangle rasterization now becomes
a lot simpler. In the `drawTriangle` method we first compute the triangle
equations and discard back-facing triangles. We compute the triangle bounding box
and clip it to the scissor rectangle as we did in [Part 1]({% post_url
2017-06-06-developing-a-software-renderer-part1 %}).

```cpp
void drawTriangle(const Vertex& v0, const Vertex &v1, const Vertex &v2)
{
  // Compute triangle equations.

  TriangleEquations eqn(v0, v1, v2);

  // Check if triangle is back-facing.

  if (eqn.area < 0)
    return;

  // Compute triangle bounding box and clip to scissor rect.

  [...]

```

Then we round the bounding box values to a block grid. This allows us to then
easily step whole blocks in the following for loops which step the x and y
values in BlockSize steps.

```cpp

  // Round to block grid.

  minX = minX & ~(BlockSize - 1);
  maxX = maxX & ~(BlockSize - 1);
  minY = minY & ~(BlockSize - 1);
  maxY = maxY & ~(BlockSize - 1);
```

Inside the for loops we compute the `EdgeData` for the four corners and test
each corner for containment in the triangle. If all are outside we can safely
reject the whole block. If all are inside we can rasterize a fully covered block
with `rasterizeBlock<false>` where the template parameter tells if the edge
equations should be computed or not. Since the block is fully covered the edge
equations will not need to be computed and we can pass false. If the block is
partially covered we call `rasterizeBlock<true>` to make that function also
compute the edge equations.

```cpp
  float s = (float)BlockSize - 1;

  // Add 0.5 to sample at pixel centers

  for (float x = minX + 0.5f, xm = maxX + 0.5f; x <= xm; x += BlockSize)
  for (float y = minY + 0.5f, ym = maxY + 0.5f; y <= ym; y += BlockSize)
  {
    // Test if block is inside or outside triangle or touches it

    EdgeData e00; e00.init(eqn, x, y);
    EdgeData e01 = e00; e01.stepY(eqn, s);
    EdgeData e10 = e00; e10.stepX(eqn, s);
    EdgeData e11 = e01; e11.stepX(eqn, s);

    int result = e00.test(eqn) + e01.test(eqn) + e10.test(eqn) + e11.test(eqn);

    // All out.

    if (result == 0)
      continue;

    if (result == 4)
      // Fully Covered

      rasterizeBlock<false>(eqn, x, y);
    else
      // Partially Covered

      rasterizeBlock<true>(eqn, x, y);
  }
}
```

Block rasterization also becomes simple with the help of our utility classes and
methods. We compute the edge and pixel values for the top-left corner of the
block and step them along the x and y direction to avoid having to compute the
whole equation per pixel.

```cpp
template <bool TestEdges>
void rasterizeBlock(const TriangleEquations &eqn, float x, float y)
{
  PixelData po;
  po.init(eqn, x, y);

  EdgeData eo;
  if (TestEdges)
    eo.init(eqn, x, y);

  for (float yy = y; yy < y + BlockSize; yy += 1.0f)
  {
    PixelData pi = po;

    EdgeData ei;
    if (TestEdges)
      ei = eo;

    for (float xx = x; xx < x + BlockSize; xx += 1.0f)
    {
      if (!TestEdges || ei.test(eqn))
      {
        int rint = (int)(pi.r * 255);
        int gint = (int)(pi.g * 255);
        int bint = (int)(pi.b * 255);
        Uint32 color = SDL_MapRGB(m_surface->format, rint, gint, bint);
        putpixel(m_surface, (int)xx, (int)yy, color);
      }

      pi.stepX(eqn);
      if (TestEdges)
        ei.stepX(eqn);
    }

    po.stepY(eqn);
    if (TestEdges)
      eo.stepY(eqn);
  }
}
```

## Conclusion

Our block based approach works and it is definitely faster then the simple
approach from the first part.

Other improvements yet to come are the support of texture coordinates and other
per vertex parameters, perspective correct parameter interpolation,
multi-threaded rasterization and a pixel shader framework that allows us to
configure the per pixel operations in a flexible manner to support texture
mapping, alpha blending and other stuff. These are topics that will be covered
in the next posts.

Continue on [Part 3]({% post_url 2017-06-15-developing-a-software-renderer-part3
%})