---
layout: post
title: Going back to low poly palettes
date: 2017-08-09
categories: [Unity, Optimization]
tags: [Unity, Blender, Lowpoly, Rendering, Optimization]  
img_path: /assets/img/post_images/2017-08-09-lowpoly-palletes
---
Low poly 3D models give a great opportunity for optimization. Let’s take a look at some of them.

![Kenney pack](001.webp){: width="800"}

A popular 3D nature pack by Kenney contains a set of two-color (mostly) low-poly models. Two-color nature works using different submeshes and different materials, which results in unneeded movement between CPU and GPU (different materials can’t be batched together). This is an example in Unity:

![Unity screenshot - 16 batches](002.webp){: width="800" }

We can see that with a standard colored material even when every object is marked as static, it only batches 32 draw calls out of 48 (while total number of objects is clearly less than 48). We still use much more draw calls than it’s really needed for this kind of objects. Also note that there are no shadows in the scene (they tend to double the number of draw calls in general)

One way to avoid this extensive usage of drawcalls is to use vertex colors. Basically each vertex is assigned with its own color and then drawn as if it was a color of material. It enables us as users to have only one material without any additional color, so all objects are batched together if they are marked as static:

![Unity screenshot - 2 batches](003.webp){: width="800" }

All objects share the same vertex color material and are batched (second batch is for Skybox)

And this way is usually presented as the most optimal. But there’s a problem. What if one wants to change the mood of the scene? The whole repainting will be needed. I suggest a simpler thing (and I’m sure it’s been done before, I just didn’t stumble upon this solution myself). We separate object UVs into tiny color islands (as tiny as a vertex can be) and then assign a small 32x32 or even 16x16 texture with small color squares for those islands:

![Blender](004.webp){: width="400" }

And then the 32x32 palette texture:

![Texture](005.webp){: width="400" }

Blank space is for different kinds of objects in the future.

And this is what we get in Unity:

![Unity](006.webp){: width="800" }

Still one batch for all of the objects. And here’s the funniest part, an autumn forest using a different palette texture:

![Unity](007.webp){: width="800" }

No need to repaint all the models, just replace the texture. It even allows for mesh data optimization — UV coordinates take two floating point values per vertex (64 bits) while Vertex Paint usually takes four (128 bits). It also allows easier dynamic batching in case of Unity. 32x32 RGB texture, compressed with ETC, takes 500 bytes of space, which is very little compared to individual vertex colors of every mesh inside a project.

Cheers!