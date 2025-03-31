---
layout: post
title: Recreating Obra Dinn style dithering
description: My attempt at recreating Obra Dinn style dithering using a cubemap
date: 2025-03-28 16:13 -0400
categories: [programming, graphics programming]
tags: [graphics programming, post processing, godot]
media_subpath: /assets/2025-03-28-obra-dinn-style-dithering
---

The Surface-Stable Fractal Dithering I implemented in my previous post looks cool, but it didn't really match the aesthetics in Obra Dinn. I wanted to try out Godot's newly released [Compositor](https://docs.godotengine.org/en/stable/tutorials/rendering/compositor.html), and this is my attempt at recreating the effect.

The code for this project can be found on [GitHub](https://github.com/tufourn/Dither-ObraDinn).

![Final cubemap](10_cubemap_alt.gif)

The first thing I did was a simple screen space to make sure I got the Compositor set up correctly. As expected, the dithering effect works well enough for static images, but when you move around, the pixels kinda swim around which is a major problem. Effect scaled up for clarity.

![Screen space](00_screenspace.gif)

Lucas Pope tried several methods, but the one that worked best was to [map the dithering pattern to a sphere around the camera](https://forums.tigsource.com/index.php?topic=40832.msg1363742#msg1363742). He mentioned hand tweaking the mapping until it looks good, but did not specify how. Instead of doing that, I wanted to try something more simple. My strategy is to project a sphere onto a cubemap and sampling that.

The first thing to do was to create a cubemap and make sure that it works. Here I am using a tiled 8x8 Bayer matrix. Again, the effect is scaled up for clarity.

![Cubemap](01_cubemap.gif)

This looks better than the initial screen space dithering. The dithering pattern stays fixed to objects as the camera rotates. But the pattern is warped at the cube corners.

![Cubemap warp](02_cubemap_warp.png)

So I tried to compensate by making the dither pattern further away from center bigger.

| ![Bayer](03_bayer_texture.png) | ![Bayer_stretch](04_bayer_texture_stretch.png) |

This is done by mapping a cube face onto a sphere.

```gdscript
var image_sphere : Image = Image.create_empty(resolution, resolution, false, Image.FORMAT_L8)
for x in range(resolution):
  for y in range(resolution):
    var cube_x : float = (float(x) / (resolution - 1) - 0.5) * 2.0
    var cube_y : float = (float(y) / (resolution - 1) - 0.5) * 2.0
    const cube_z : float = 1.0 # all 6 faces of cubemap use the same texture so we only do one face (z+)

    var dir = Vector3(cube_x, cube_y, cube_z)
    dir /= dir.length()

    var angle_x : float = atan2(dir.z, dir.x) # 3pi/4 downto pi/4
    var angle_y : float = atan2(dir.z, dir.y) # 3pi/4 downto pi/4

    # normalize to 0-1
    var angle_x_normalized = (angle_x - PI / 4) / (PI / 2)
    var angle_y_normalized = (angle_y - PI / 4) / (PI / 2)

    bayer_x = floori(angle_x_normalized * bayer_size * tiling) % bayer_size
    bayer_y = floori(angle_y_normalized * bayer_size * tiling) % bayer_size
    # when angle_normalized == 1, bayer index should be (bayer_size - 1)
    # but the modulo calculation returns 0 so we have to fix it
    if (x == 0):
      bayer_x = bayer_size - 1
    if (y == 0):
      bayer_y = bayer_size - 1

    var bayer_val = bayer[bayer_x][bayer_y]
    image_sphere.set_pixel(resolution - 1 - x, resolution - 1 - y, Color(bayer_val, bayer_val, bayer_val, 1.0))
```

![Stretch](05_stretch_warp.png)

It looks a bit better now, the dither pattern at the corners now looks more uniform than before. But then I ran into another problem: The Bayer dither pattern doesn't really tile well in a cubemap. It's more apparent when you apply the effect. Because the dither pattern at the corners are larger now, the seams look even more noticeable.

![Seam](06_seam.png)

After messing around with the mapping a bit, I ended up with this:

```gdscript
var bayer_x : int = roundi(angle_x_normalized * bayer_size * tiling) % bayer_size
var bayer_y : int = roundi(angle_y_normalized * bayer_size * tiling) % bayer_size
```

| ![Bayer](03_bayer_texture.png) | ![Bayer_stretch](07_bayer_stretch_alt.png) |

It still doesn't tile perfectly, but it looks a lot better than before. The seams are a lot less noticeable.

![Warp alt](09_stretch_wrap_alt.png)

![Seam alt](08_seam_alt.png)

![Final cubemap](10_cubemap_alt.gif)

The dither pattern are spaced reasonably uniformly. There are probably better ways to do this, but for me this is good enough.
