---
layout: post
title: World Space MatCap Shading
date: 2017-09-16
categories: [Unity, Shaders]
tags: [Unity, Rendering, Shaders, Matcaps, Tutorial]  
img_path: /assets/img/post_images/2017-09-16-matcaps
---

> This article was originally written in 2017 and doesn't represent the idiomatic usage of MatCaps. Right now I would have probably used spherical harmonics for the same purpose. However, article may still be useful.
{: .prompt-info }


In our racing/runner game we decided to use matcap shading (at least for now)

MatCap (stands for Material Capture) does what it sounds like — captures a material properties like shininess / roughness, overall “feel” of the texture. It’s usually used just for that purpose — to create a feeling of some material at low rendering cost. MatCaps are usually associated with ZBrush and you can find a lot of MatCap examples on their site.

![matcap example](001.webp){: width="800"}
_<https://pixologic.com/zbrush/downloadcenter/library/#prettyPhoto>_

The cost of rendering is low because the whole material is just one texture sample away from rendering it on screen. All hard and expensive calculations are done “offline” while rendering the sphere itself.

As it’s been said, MatCaps are usually used for the material texture “feel”. But it’s also possible to use them just for lighting purposes only. We just need to capture our scene lighting on a non-reflective white sphere and then use that sphere to light any dynamic object on the scene. Much like in Spherical Harmonics, MatCaps rendering speed does not in any way depend on a number of light sources or anything else.

As we are making this lighting in Unity, a lot of work has already been done by other people. There’s a set of free MatCap shaders here: <https://www.assetstore.unity3d.com/en/#!/content/8221>

But there’s one problem — those shaders don’t use world-space normals for the MatCap sampling. The sampling depends soloely on viewport angle and position. It means that lighting depends on Camera rotation, not on Object rotation. Have a look:

![matcap example](002.webp){: width="550"}
_Initial camera rotation_

![matcap example](003.webp){: width="550"}
_New camera rotation_

Just by rotating camera we change lighting conditions. Lighting doesn’t really work like that. Basically this setup is OK for a static angle camera like some top-down RPG, or any other game where the camera doesn’t rotate.

But in our case there **are** camera rotations in some cases. So we need to be able to use world-space normals instead of the view-space normals. And this is how it looks like:

![matcap example](004.webp){: width="450"}
_Rendered sphere with a MatCap_

![matcap example](005.webp){: width="450"}
_The MatCap itself_

And if we rotate our camera around a sphere, we see this:
![matcap example](006.webp){: width="450"}
_Rotating around the sphere_

The lighting remains the same — directions of whites and blacks remain the same.

While making changes to shaders I also noticed some artifacts along the “50% seam” where texture reads start to sample the very edge of the sphere:
![matcap example](007.webp){: width="800"}

I can think of two possible solutions:

1. Texture Bleed along the edge of the sphere, so each pixel bleeds further into texture, reducing this white-brown edge to a zero
2. Reduce the radius of MatCap texture sampling a bit

First solution is preferrable because it will allow to preserve these edge details. Second solution though is more simple and straightforward, so I decided to try that first:

![matcap example](008.webp){: width="400"}
_No artifacts — lost details_

As you can see we have lost that flare detail, but got rid of the artifacts.

And this is how the in-game car looks without and with MatCap lighting:

![matcap example](009.webp){: width="800"}
_Unlit VS MatCap lighting_

![matcap example](010.webp){: width="400"}
_Lighting matcap_

The shader can be found at <https://github.com/KumoKairo/Worldspace-Normal-Lighting/blob/master/MatCap_Plain.shader>