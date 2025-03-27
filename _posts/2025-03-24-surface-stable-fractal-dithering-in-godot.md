---
title: Surface-Stable Fractal Dithering in Godot
description: Implementation of runevision's Surface-Stable Fractal Dithering in Godot
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

But UV mapping the dither pattern to the geometry results in uneven scaling. This technique compensates for it by scaling down the pattern when it becomes too large and scaling up the pattern when it becomes too small. This is done by using the derivatives of the UV, or "how fast" the UV is changing, in screen space. Here they are, visualized as `abs(dFdx(UV)) * 100` and `abs(dFdy(UV) * 100)`, respectively.

| ![dFdx](03_dfdx.png) | ![dFdy](04_dfdy.png) |

Given these derivative values, the technique uses singular value decomposition to find the maximum and minimum frequencies. Here they are, scaled up by 10x.

| ![Max freq](05_freqmax.png) | ![Min freq](06_freqmin.png) |

The minimum frequency, combined with the material properties, are used to calculate the base dither dot spacing value.

```glsl
// freq is vec2(max_freq, min_freq)
// spacing variable which correlates with average distance between dots
float spacing = freq.y;

// scale by specified input scale
float scale_exp = exp2(scale);
spacing *= scale_exp;
```

By scaling down by half when the pattern becomes twice as large and vice versa, we can adjust the scaling so that patterns are scaled roughly the same size on the screen, varying only up to a factor of two. I'm using a simple dot as the texture here to visualize the effect.

![Derivative UV](11_derivative_uv.png)

We can see that a dot kind of splits into 4 smaller dots when it crosses the threshold to the lower fractal level. By shifting the dot center to the corner of the texture, we can make it so that it appears like the big dot becomes smaller and is joined by 3 more dots surrounding it when the scaling changes.

| ![Dot](13_dot_tex.png) | ![Dot cornered](14_dot_corner_tex.png) |

![Corner dot](12_corner_dot.png)

To reduce the abruptness when the scaling changes, we can make the new dots appear one by one instead of all at once. This is achieved with a 3D texture, where each layer contains one additional dot compared to the previous layer.

The 4x4 radial gradient dither pattern (64x64x8) with the layers placed side-by-side looks like this, with the leftmost layer containing 1 dot and the rightmost layer containing 16 dots.

![3D texture](08_3dtex.png)

The mesh UV is divided by the nearest lower power of two of the spacing to get the adjusted UV to sample the 3D texture. By scaling UV by brightness, we are controlling brightness by changing both dot spacing and dot sizes.

```glsl
// keep the spacing the same regardless of pattern
spacing *= dots_per_side * 0.125;

float brightness_spacing_multiplier = pow(brightness_curve * 2.0 + 0.001, -1.0);
spacing *= brightness_spacing_multiplier;

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

Next, the contrast factor is calculated based on the spacing, ratio of the UV frequencies, and the material properties to get sharp dots from the radial gradient pattern.

```glsl
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
float threshold = 1.0 - brightness;
```

By setting the threshold to be `1.0 - brightness`, the brightness is controlled by varying dot sizes. This cancels out the effect from scaling the UV with brightness and we end up with brightness controlled only by modifying dot spacing.

Then the final color is calculated and we get our result.

```glsl
// get pattern value relative to threshold, scale by contrast and add base value
float bw = clamp((pattern - threshold) * dot_contrast + base_val, 0.0, 1.0);
```

![Screenshot](00_screenshot.png)

## Godot implementation

My implementation of the technique in Godot can be found on [GitHub](https://github.com/tufourn/Dither3D-Godot).

The main logic is in `Shaders/Dither3D.gdshaderinc`{: .filepath} which is basically the original implementation ported to Godot's shading language. The 3D dither pattern texture files are available, but I've also included the `CreateDitherTextures.gd`{: .filepath} script which you can use to generate them yourself.

