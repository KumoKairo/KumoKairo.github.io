---
layout: post
title: Simple procedural skybox with a moon
date: 2017-08-11
categories: [Unity, Effects]
tags: [Unity, Rendering, Effects, Skybox, Tutorial]  
img_path: /assets/img/post_images/2017-08-11-skybox
---

We’re making a game and were looking for some references. This one by Mark Kirkpatrick seemed very interesting:

![Mark Kirkpatrick art](001.webp){: width="800"}

I tried to recreate that in Unity and here’s what we got:

![Unity art](002.webp){: width="800"}

I used standard procedural skybox shader and this set of shaders (<https://github.com/keijiro/UnitySkyboxShaders>) as a reference, but simplified it a lot. Here’s how it works.

## Gradient

We have top and bottom colors and we need to blend them together. Skybox shaders are interesting, there’s a set of vertices that are rendered using texture coordinates and positions. We will use texture coordinates in our case. Here’s the vertex shader:

```glsl
struct appdata
{
    float4 position : POSITION;
    float3 texcoord : TEXCOORD0;
};

struct v2f
{
    float4 position : SV_POSITION;
    float3 texcoord : TEXCOORD0;
};

v2f vert(appdata v)
{
    v2f o;
    o.position = UnityObjectToClipPos(v.position);
    o.texcoord = v.texcoord;
    return o;
}
```

We just pass texture coordinates without any changes. As you can see, those texture coordinates are stored in a 3D vector. That vector represents a unit vector of direction towards the point on a skybox (much like a directional light rotation `_WorldSpaceLightPos0` used in shaders. So Y component of that vector represents vertical direction of that point and we can use that to blend two colors together. All components change from -1 to 1 (Y component is -1 at the bottom and 1 at the top). If we normalize the Y component and blend our two colors together, we get a very smooth transition:

```glsl
alf4 frag(v2f i) : COLOR
{
    half p = i.texcoord.y;
    
    // Moving [-1, 1] range to [0, 1]
    half normP = p * 0.5 + 0.5;
    half3 color = lerp(_GroundColor,_SkyTint, p);;
    return half4(color, 1.0);
}
```

![Unity art](003.webp){: width="800"}

So we need a way to move the border closer to the center and squish it a bit.

```glsl
half4 frag(v2f i) : COLOR
{
    half p = i.texcoord.y;

    float p1 = pow(min(1.0f, 1.0f - p), _Exponent);

    half3 color = lerp(_GroundColor, _SkyTint, p1);
}
```

We subtract current Y component from 1 and take a minimum of the subtraction and 1.0. As values are less than 0 after we cross a horizontal plane, subtraction absolute values will be more than 1, and this is where we will blend the Ground color. `_Exponent` parameter allows to tweak “hardness” of the gradient transition. This is how it looks with `_Exponent = 15`:

![Unity art](004.webp){: width="800"}

## Moon

The moon was a bit tricky and got me digging a lot of Unity shader sources. The idea itself is simple though.

1. Set our moon center as a directional unit vector (just like the directional `_WorldSpaceLightPos0` is stored)
2. Find distance between the moon center directional vector and a skybox directional vector (our `float3 textureCoords`)
3. Use that distance to calculate falloff of the moon circle using some function that allows to tweak the hardness of the moon edges.

Here’s a modified `calcSunSpot` function from the standard procedural skybox:

```glsl
#define HARDNESS_EXPONENT_BASE 0.125

half calcSunSpot(half3 sunDirPos, half3 skyDirPos)
{
    half3 delta = sunDirPos - skyDirPos;
    half dist = length(delta);
    half spot = 1.0 - smoothstep(0.0, _SunSize, dist);
    return 1.0 - pow(HARDNESS_EXPONENT_BASE, spot * _SunHardness);
}
```

5 and 6 lines calculate distance between two directional vectors (how far is the fragment from the center of the moon)

7 line calculates the circular form of our moon. We take advantage of the smoothstep function clamping abilities outside of the interpolation range. Subtracting it from 1 gives us bigger values in the center (whites)

![Unity art](005.webp){: width="500"}
_Smoothstepped moon without exponential squishing logic_

The 8 line squishes our gradient transition. You can imitate it’s mathematical logic in Photoshop using Curves adjustments:

![Unity art](006.webp){: width="500"}
![Unity art](007.webp){: width="500"}
_Moon inside Unity_

## All together

We need to blend our gradient with the moon now. We will also need our ground fog to affect the moon.

```glsl
half4 frag(v2f i) : COLOR
{
    half p = i.texcoord.y;

    float p1 = pow(min(1.0f, 1.0f - p), _Exponent);
    float p2 = 1.0f - p1;

    // Our moon circle
    half3 mie = calcSunSpot(_SunPosition.xyz, i.texcoord.xyz) * _SunColor;
    // Blend ground fog with the moon
    half3 col = _GroundColor * p1 + mie * p2;
    // Add sky color
    col += _SkyTint * p2;

    return half4(col, 1.0);
}
```

Line 9: calculating and coloring our moon (sun) spot
Line 11: blending the ground color with the moon
Line 13: adding the top (sky) color

This is what it looks like in the end (I changed the color of the moon from white to light cyan):

![Unity art](008.webp){: width="700"}

One thing to mention — `_SunPosition` should be a unit vector (with a magnitude of 1). It’s tricky to set it by hand, so you can either use some helper methods to calculate a unit vector from rotation, or use one directional light on the scene just for its rotation (in this case either your lighting will depend on it’s rotation or you’ll need to use unlit shaders everywhere).

Here’s a full shader source. Create a new material and set it as skybox inside Unity’s lighting preferences window — <https://github.com/KumoKairo/Procedural-Skybox/blob/master/GradientMoonSkybox.shader>

Cheers.