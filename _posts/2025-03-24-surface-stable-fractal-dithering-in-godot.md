---
title: Surface-Stable Fractal Dithering in Godot
description: Godot implementation of runevision's Surface-Stable Fractal Dithering
date: 2025-03-24 18:03 -0400
categories: [programming, graphics programming]
tags: [graphics programming, godot]
media_subpath: /assets/2025-03-24-surface-stable-fractal-dithering-in-godot
---

About a month ago, a friend recommended me [Return of the Obra Dinn](https://store.steampowered.com/app/653530/Return_of_the_Obra_Dinn/). I haven't finished the game yet, but I thought the dithering effect looked super cool. I also saw runevision's [Surface-Stable Fractal Dithering](https://www.youtube.com/watch?v=HPqGaIMVuLs) video which inspired me to try implementing the technique.

I've been wanting to learn Godot for some time now, so I figured this would be a good opportunity to give it a go.

![Screenshot](00_screenshot.png)

## How it works

If you haven't watched runevision's video already, you should go watch it. It's a great video.

Here's an overview of the technique. The inputs to the dithering algorithm are:
- brightness
- geometry texture coordinates (UV)
- derivatives of the UVs in screen space.

The brightness is just the fragment color converted to grayscale.

```gdscript
float get_grayscale(vec4 color) {
    return clamp(0.299 * color.r + 0.587 * color.g + 0.114 * color.b, 0.0, 1.0);
}
```

![Grayscale](01_grayscale.png)

The geometry texture coordinates is just the `UV` value passed from the vertex shader. By mapping the dither pattern to the texel space of the scene geometry, the pattern can "stick" to surfaces.

![UV](02_uv.png)

But mapping the dither pattern to the geometry using UV mapping directly results in the pattern sometimes being scaled too small or too large. To scale the dither pattern so it's roughly constant in size on the screen, the derivatives of the UV in screen space are also needed. Here they are, visualized as `abs(dFdx(UV)) * 100` and `abs(dFdy(UV) * 100)`, respectively. Whenever the pattern becomes two times too large, we halve the scale and vice versa.

| ![dFdx](03_dfdx.png) | ![dFdy](04_dfdy.png) |

Given these derivative values, the technique uses singular value decomposition to find the maximum and minimum frequencies. Here they are, scaled up by 10x. The minimum frequency, combined with the material properties, are used to calculate the base dither dot spacing value.

| ![Max freq](05_freqmax.png) | ![Min freq](06_freqmin.png) |

We also need to prepare the 3D texture for our radial gradient dither pattern. The 4x4 dither pattern with the layers placed side-by-side looks like this:

![3D texture](08_3dtex.png)

The mesh UV is divided by the nearest lower power of two of the spacing to get the adjusted UV to sample the 3D texture.

```glsl
float spacing_log = log2(spacing);
float pattern_scale_level = floor(spacing_log);
vec2 uv_dither = uv / exp2(pattern_scale_level);
```

The remaining fraction is used to determine which sublayer to use.

```glsl
// first layer is the one that has 1/4 of the dots
// last layer is the one with all the dots
float frac = spacing_log - pattern_scale_level;
float sub_layer = mix(0.25 * z_res, z_res, 1.0 - frac);

// texels are half a texel off from texture border
// subtract half a texel and normalize to 0-1 range
sub_layer = (sub_layer - 0.5) * inv_z_res;
```

The adjusted UV and sublayer is visualized below.

![Adjusted UV](10_adjusted_uv.png)

Then the 3D dither texture is sampled.

```glsl
float pattern = texture(dither_tex, vec3(uv_dither, sub_layer)).r;
```

![Sampled texture](07_sampled.png)

Next, the contrast factor is calculated based on the spacing, ratio of the UV frequencies, and the material properties.

```glsl
// create sharp dots from radial gradient textures by increasing contrast
float dot_contrast = contrast * scale_exp * brightness_spacing_multiplier * 0.1;

// contrast is based on the highest frequency to avoid aliasing
// scale contrast by ratio of smallest frequency and highest frequency
// adjust compensation with stretch_smoothness factor
dot_contrast *= pow(freq.y / freq.x, stretch_smoothness);
```

The base value that the contrast is scaled around is normally 0.5, but if the pattern is blurry then the brightness everywhere would be close to 0.5. To fix this, we linearly interpolate towards the base value of the original brightness as the contrast decreases.

```glsl
// this specific formula is taken from the original implementation
float base_val = mix(0.5, brightness, clamp(1.05 / (1.0 + dot_contrast), 0.0, 1.0));
```

The threshold value to compare with the radial gradient is calculated based on the brightness.

```glsl
// the brighter we want the output, the lower the threshold we need to use
float threshold = 1.0 - brightness_curve;
```

Then the final color is calculated and we get our result.

```glsl
// get pattern value relative to threshold, scale by contrast and add base value
float bw = clamp((pattern - threshold) * dot_contrast + base_val, 0.0, 1.0);
```

![Screenshot](00_screenshot.png)

## Godot implementation

My implementation of the technique in Godot can be found on [GitHub](https://github.com/tufourn/Dither3D-Godot).

The main logic is in `Shaders/Dither3D.gdshaderinc`{: .filepath} which is basically the original implementation ported to Godot's shading language. The 3D dither pattern texture files are available, but I've also included the `CreateDitherTextures.gd`{: .filepath} script which you can use to generate them yourself.

Make sure that you set `render_mode ambient_light_disabled`, or you'll end up with something like this.

![Ambient](09_ambient.png)

Because Godot doesn't provide a way to apply shading after all the lights have been applied, I had to use this trick in the `light()` shader.

```glsl
void light() {
    float NoL = clamp(dot(NORMAL, LIGHT), 0.0, 1.0);
	
    // using DIFFUSE_LIGHT as a spare variable to accumulate light
    DIFFUSE_LIGHT += LIGHT_COLOR * ATTENUATION * NoL;

    // calculate dither
    vec2 dx = dFdx(UV);
    vec2 dy = dFdy(UV);
    SPECULAR_LIGHT = get_dither_3d_color(UV, dx, dy, vec4(ALBEDO * DIFFUSE_LIGHT, 1.0)).rgb;

    // cancel contribution of DIFFUSE_LIGHT
    SPECULAR_LIGHT -= DIFFUSE_LIGHT * ALBEDO;
}
```
{: file='DitherOpaque.gdshader'}
