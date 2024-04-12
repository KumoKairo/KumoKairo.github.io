---
layout: post
title: How Ren'Py handles text rendering.
date: 2024-01-23
categories: [RenPy]
tags: [RenPy, Rendering]  
img_path: /assets/img/post_images/2024-01-23-renpy-text
---
I recently participated in a visual novel game jam where participants were required to create a complete project in 10 days. Most of the projects were created with Ren’Py - arguably the most popular visual novel engine. One thing that caught my eye was the text quality in those projects - it usually was super crisp with just right amount of anti-aliasing.

![text example](001.webp){:width="800" }

However, while reading one of the novels, I noticed some strange artifacts:

![text example](002.webp){:width="350" }

Noticed the strange black dots below the first line of text? It’s actually just cut pieces from the second line of text:

![text example](003.webp){:width="450" }

Seeing this was super strange. It could only mean that Ren’Py handles text rendering in a very different way than pretty much everyone else in the gamedev world. Text in visual novels usually appears gradually, much like typing, and there are different ways to achieve this. Apparently, Ren’Py uses some kind of masking over a pre-rendered texture containing the whole text block (let’s call it a “phrase”).

Anyway, I tried searching for an in-depth overview of Ren’Py text rendering tech, but haven’t found anything related to it other than other “bug reports” with similar screenshots from other people and a line in a documentation saying “this is a known limitation that is totally staying, and the only way to mitigate it is to increase the line spacing” ([source](https://www.renpy.org/doc/html/text.html#slow-text-concerns)).

So I decided to dig deeper into it to understand what’s going on under the hood (and also to check the tech behind super-crisp text rendering).
How it’s usually done.

Here’s a simplified process of text rendering in most games / game engines:

1. Generate a font atlas texture with needed glyphs.
2. Assign a quad to each glyph.
3. For each specific string of text, lay out the appropriate quads with needed glyphs and render them (one by one or in a single batch)

This is, of course, an incredibly simplified description of the process, but it’s more or less the same in most of the modern game engines:

Unity:

![text example](004.webp){:width="800" }

Unity’s generated font atlases - top is for the regular text, bottom is for TextMeshPro (which uses SDF and out of scope of this article).

![text example](005.webp){:width="450" }

Unreal Engine:

![text example](006.webp){:width="450" }

Godot:

![text example](007.webp){:width="450" }

You get the idea. Each glyph takes a quad, each quad refers to a portion of the font atlas texture. Phrases are rendered in one batch - all quads appear on screen during one draw call. Need more text - generate more quads, lay them out appropriately and send them to draw.

Now here’s how Ren’Py renders text:

![text example](008.webp){:width="800" }

Not only it renders the whole phrase as one “quad”, but it also has additional geometry (what is that additional thin rectangle in the middle?). If we check currently bound texture we see that we have the whole phrase itself:

![text example](009.webp){:width="800" }

If we check the frame during the appearance animation to see the geometry, we can see that indeed, it partially covers the line below:

![text example](010.webp){:width="400" }

(Red line for geometry debug covers the black bits).

Well, interesting. If we are just rendering pre-defined text that is already stored in a texture, where does it come from in the first place? I had to do some hacky things to capture the moment of text generation, but here’s how it works in Ren’Py.

## How Ren’Py generates “phrase” textures.

Each time a phrase appears on screen, Ren’Py:

1. Generates two textures (let’s name them Texture_1 and Texture_2)

2. Uploads predefined pixel data to Texture_1 (line 12 - we are sending our phrase as 919 552 bytes worth of texture data to a buffer, which is then copied as texture data).

![text example](011.webp){:width="800" }

3. Renders a quad with Texture_1 to an off-screen FBO. (basically just temporarily renders Texture_1).

![text example](012.webp){:width="800" }

4. Copies FBO contents with just rendered quad with Texture_1 to Texture_2.

![text example](013.webp){:width="800" }

5. Deletes Texture_1 and all data associated with it. We now have Texture_2 with the original phrase.

![text example](014.webp){:width="800" }

[Looking through source](https://github.com/renpy/renpy/blob/3a3d271404d06178302a1a78208f1a114be0dbfa/renpy/gl2/gl2texture.pyx#L423) code shed some light on reasons behind this - to premultiply alpha values for the text. However, in my byte-to-byte tests both textures turned out to be exactly the same.

![text example](015.webp){:width="800" }
_Byte comparison between two textures_

 Anyway, after this copying-rendering process is done, this Texture_2 is then used until we want to update our text. At that moment, the process repeats.

However, the main question remains - where does this text texture data come from in the first place. The engine still needs to lay out the glyphs, “render” them as texture data and then pass as buffer data to our Texture_1.

After digging through some more RenPy source code, I figured the following:

1. It relies heavily on PyGame’s Blit functionality.
2. Each Font glyph is a PyGame Surface (which is basically a separate image with some additional data).
3. When setting text and after the layout phase, each character is blitted into a “phrase” texture (in our case - what will happen to be Texture_1).

Step 3 should be pretty expensive for a large number of characters, and I couldn’t catch this process using the frame debugger app (maybe it happens between frame delimiters so it just drops them). This must be the reason why Ren’Py tries to cache the “phrase” into a texture and use it over and over again until it’s time to change the text.

I wish I had more time to dig into more source code, but I have a pretty clear picture in my mind now.

## A few closing notes.

Is it possible to fix the text artifact? Absolutely, but Ren’Py will have to switch to quad-based rendering as opposed to cached-texture based rendering. Judging by the code I have seen, it’s not a trivial task. I can speculate about the reasons behind this decision - PyGame seems to be quite blit-oriented, encouraging this process of “copying textures over textures”, so this way of rendering text was the easiest to do. 