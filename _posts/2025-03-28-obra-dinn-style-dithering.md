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

The first thing I did was a simple screen space dithering to make sure I got the Compositor set up correctly. As expected, the dithering effect works well enough for static images, but when you move around, the pixels kinda swim around which is a major problem. Effect scaled up for clarity.

![Screen space](00_screenspace.gif)

Lucas Pope tried several methods, but the one that worked best was to [map the dithering pattern to a sphere around the camera](https://forums.tigsource.com/index.php?topic=40832.msg1363742#msg1363742). He mentioned hand tweaking the mapping until it looks good, but did not specify how. Instead of doing that, I wanted to try something more simple. My strategy is to project a sphere onto a cubemap and sampling that.

First we create the cubemap texture.

```gdscript
var bayer : CompressedTexture2D = load("res://Dither/cubemap_face.png")
var bayer_img : Image = bayer.get_image()
bayer_img.convert(Image.FORMAT_R8)

var bayer_img_data : PackedByteArray = bayer_img.get_data()
var cubemap_data : Array[PackedByteArray] = []
for i in range(6):
  cubemap_data.push_back(bayer_img_data)

var cubemap_format : RDTextureFormat = RDTextureFormat.new()
cubemap_format.format = RenderingDevice.DATA_FORMAT_R8_UNORM
cubemap_format.texture_type = RenderingDevice.TEXTURE_TYPE_CUBE
cubemap_format.usage_bits = RenderingDevice.TEXTURE_USAGE_SAMPLING_BIT
cubemap_format.array_layers = 6
cubemap_format.width = bayer.get_width()
cubemap_format.height = bayer.get_height()

var cubemap_view : RDTextureView = RDTextureView.new()

cubemap_texture = rd.texture_create(cubemap_format, cubemap_view, cubemap_data)
```
{:file="post_process.gd"}

Then we need to calculate the direction from the eye to each pixel. This is done by transforming the frustum corners from NDC (normalized device coordinates) back to world space, and subtracting the position of the eye.

```gdscript
var projection : Projection = render_scene_data.get_view_projection(view)
var view_inverse : Transform3D = render_scene_data.get_cam_transform()
var eye_offset : Vector3 = render_scene_data.get_view_eye_offset(view)
var eye_position : Vector3 = view_inverse * Vector3(0.0, 0.0, 0.0) + eye_offset

var frustum_corners_ndc : Array[Vector4] = [
  Vector4(-1, -1, 1, 1), Vector4(1, -1, 1, 1), # TL, TR
  Vector4(-1, 1, 1, 1), Vector4(1, 1, 1, 1) # BL, BR
]

var frustum_corners : PackedVector4Array
for ndc in frustum_corners_ndc:
  var view_space = projection.inverse() * ndc
  var corner = view_inverse * Vector3(view_space.x, view_space.y, view_space.z) - eye_position
  frustum_corners.append(Vector4(corner.x, corner.y, corner.z, 1.0))
```
{:file="post_process.gd"}

And lerp (linear interpolation) in the compute shader based on the position of each pixel on the image.

```glsl
layout(push_constant, std430) uniform PushConstants {
  vec4 frustum_top_left;
  vec4 frustum_top_right;
  vec4 frustum_bottom_left;
  vec4 frustum_bottom_right;
  vec2 raster_size;
  vec2 reserved;
} pc;

vec3 get_direction_to_pixel(vec2 uv) {
  float u = float(uv.x) / (pc.raster_size.x - 1);
  float v = float(uv.y) / (pc.raster_size.y - 1);

  vec3 top = mix(pc.frustum_top_left.xyz, pc.frustum_top_right.xyz, u);
  vec3 bot = mix(pc.frustum_bottom_left.xyz, pc.frustum_bottom_right.xyz, u);
  vec3 direction = mix(top, bot, v);

  return normalize(direction);
}
```
{:file="dither.glsl"}

Then we can sample the cubemap and apply the threshold. Here I am using a tiled 8x8 Bayer matrix as the dither texture. Again, the effect is scaled up for clarity.

```glsl
vec4 color = imageLoad(color_image, uv);

float lum = color.r * 0.2125 + color.g * 0.7154 + color.b * 0.0721;
float bayer_threshold = texture(cubemap_image, get_direction_to_pixel(uv)).r;
color.rgb = vec3(step(bayer_threshold, lum));

imageStore(color_image, uv, color);
```
{:file="dither.glsl"}

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
{:file="create_cubemap_image.gd"}

![Stretch](05_stretch_warp.png)

It looks better now, the dither pattern at the corners now looks more uniform than before. But then I ran into another problem: The Bayer dither pattern doesn't really tile well in a cubemap. It's more apparent when you apply the effect. Because the dither pattern at the corners are larger now, the seams look even more noticeable.

![Seam](06_seam.png)

After a bit of messing around with the mapping, I ended up with this:

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
