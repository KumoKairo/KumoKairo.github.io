---
layout: post
title: Understanding and implementing regular Matcaps in Unity
date: 2023-05-30
categories: [Unity, Shaders]
tags: [Unity, Rendering, Shaders, Matcaps, Tutorial]  
img_path: /assets/img/post_images/2023-05-30-matcaps
---

I recently wanted to refresh my knowledge and understanding of matcaps for one of my current projects. Turned out that [one amazing github repository](https://github.com/nidorx/matcaps) with matcap textures and reference implementations has a link to my previous article on matcaps. I also noticed that two other Unity reference links are broken (_rip wiki.unity3d_), and my previous article doesn’t really use matcaps the intended way. So I felt the urge to go back and write an actual explanation of how it’s all working.

## Refresher

To refresh you on matcaps - it’s basically a very complicated material represented as a square texture showing one side of a sphere:

![matcap example](001.webp){:width="400" }
_Courtesy of <https://github.com/nidorx/matcaps>_

Which is then if applied to a complex 3D object correctly, can yield complex-looking results at the cost of one texture lookup:

![matcap example](002.webp){: width="800"}
_Glare and rim lights move when viewing angles change_

It does not react to lights, can’t have areas of varying roughness or reflectiveness and can’t have a 360 reflection view.

## Intended way

Matcaps are supposed to work solely on relation between object normals and view space. After conversion, these normals are in [-1 : 1] range, and kinda become 2D. We then just need to scale this [-1 : 1] range to our regular [0 : 1] range of UV coordinates. These UVs are then used directly to sample the matcap texture.

The picture below tries to illustrate this. Note that:
1. We only care about X and Y coordinates
2. We intentionally “lose” the depth of the Z coordinate and vectors stop being normalized.

![matcap normals](003.webp){: width="500"}

You can do the rest of the math — to convert any value from [-1 : 1] range to [0 : 1] you just need to multiply it by 0.5 (to move a value to [-0.5 : -0.5] range) and then just add 0.5 to it. Try doing this for the picture above, you will see that you are getting sensible values for each of the three presented vectors.

## Let’s code

Start with an “Unlit” shader dummy in Unity (Create → Shader → Unlit Shader) and delete everything related to Fog and Texture Sampling (leaving only the sampler2D). You also won’t need model UVs in the appdata structure. Add a NORMAL to your appdata structure instead.

![create new shader](004.webp){: width="700"}

```glsl
Shader "Unlit/MatcapExample"
{
    Properties
    {
        _MainTex ("Texture", 2D) = "white" {}
    }
    SubShader
    {
        Tags { "RenderType"="Opaque" }

        Pass
        {
            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag

            #include "UnityCG.cginc"

            struct appdata
            {
                float4 vertex : POSITION;
                float3 normal : NORMAL;
            };

            struct v2f
            {
                float2 uv : TEXCOORD0;
                float4 vertex : SV_POSITION;
            };

            sampler2D _MainTex;

            v2f vert (appdata v)
            {
                v2f o;
                o.vertex = UnityObjectToClipPos(v.vertex);
                return o;
            }

            fixed4 frag (v2f i) : SV_Target
            {
                fixed4 col = tex2D(_MainTex, i.uv);
                return col;
            }
            ENDCG
        }
    }
}
```

We just need to convert our object-space normals to world-view by using matrix multiplication in our vertex shader and then adjust the range:

```glsl
v2f vert (appdata v)
{
...
  // Converting to View-Model space
  float2 normalVM = mul(UNITY_MATRIX_MV, v.normal).xy;
  // Adjusting the range from [-1 : 1] to [0 : 1]
  float2 uvs = normalVM * 0.5 + float2(0.5, 0.5);
  o.uv = uvs;
...
}
```

You can now use that `o.uv` in the fragment shader to sample your matcap texture.

Material preview should look exactly like the matcap texture:

![create new shader](005.webp){: width="800"}

A few notes on the code — you don’t need to normalize the normals because everything is calculated in the vertex shader. Input normals are usually normalized by the engine or the modellers and you usually only need to normalize the interpolated values when passing from vertex to fragment shaders.

You can move View transform to the fragment shader if you are getting geometry-induced artifacts. In this case, separate one matrix multiplication to two: one multiplication of `UNITY_MATRIX_M` in vertex shader, and another multiplication of `UNITY_MATRIX_V` in fragment shader. In this case make sure to normalize the interpolated normal. Both versions of the shader are below.

Mostly-vertex shader (optimized).

```glsl
hader "Unlit/MatcapExample"
{
    Properties
    {
        _MainTex ("Texture", 2D) = "white" {}
    }
    SubShader
    {
        Tags { "RenderType"="Opaque" }

        Pass
        {
            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag

            #include "UnityCG.cginc"

            struct appdata
            {
                float4 vertex : POSITION;
                float3 normal : NORMAL;
            };

            struct v2f
            {
                float2 uv : TEXCOORD0;
                float4 vertex : SV_POSITION;
            };

            sampler2D _MainTex;

            v2f vert (appdata v)
            {
                v2f o;
                o.vertex = UnityObjectToClipPos(v.vertex);
                float2 normalVM = mul(UNITY_MATRIX_MV, v.normal).xy;
                float2 uvs = normalVM * 0.5 + float2(0.5, 0.5);
                o.uv = uvs;
                return o;
            }

            fixed4 frag (v2f i) : SV_Target
            {
                fixed4 col = tex2D(_MainTex, i.uv);
                return col;
            }
            ENDCG
        }
    }
}
```

Mostly-fragment shader (noticeably heavier, but **probably** less artifacts)

```glsl
Shader "Unlit/MatcapExample"
{
    Properties
    {
        _MainTex ("Texture", 2D) = "white" {}
    }
    SubShader
    {
        Tags { "RenderType"="Opaque" }

        Pass
        {
            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag

            #include "UnityCG.cginc"

            struct appdata
            {
                float4 vertex : POSITION;
                float3 normal : NORMAL;
            };

            struct v2f
            {
                float3 normal : NORMAL;
                float4 vertex : SV_POSITION;
            };

            sampler2D _MainTex;

            v2f vert (appdata v)
            {
                v2f o;
                o.vertex = UnityObjectToClipPos(v.vertex);
                o.normal = mul(UNITY_MATRIX_M, v.normal);
                return o;
            }

            fixed4 frag (v2f i) : SV_Target
            {
                float2 uvs = mul(UNITY_MATRIX_V, i.normal).xy * 0.5 + float2(0.5, 0.5);
                fixed4 col = tex2D(_MainTex, uvs);
                return col;
            }
            ENDCG
        }
    }
}
```

Hope Matcaps are easier to understand now. To me they are certainly an example of people’s ingenuity.