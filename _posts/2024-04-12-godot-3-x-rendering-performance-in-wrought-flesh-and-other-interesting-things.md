---
layout: post
title: Godot 3.x rendering performance in Wrought Flesh. And other interesting things.
date: 2023-12-25
categories: [Godot, Optimization]
tags: [Godot, Rendering, Optimization, Profiling]  
img_path: /assets/img/post_images/2023-12-25-wrought-flesh-first
---
> TL;DR of this article is: I wasn't able to find out why Wrought Flesh lags, but it's definitely not because of Rendering / Shaders / GPU. Although it doesn't give a definitive answer, the article can still be considered useful. I am currently working on another Wrought Flesh performance article, expect updates.
{: .prompt-info }

A recent Reddit post regarding 3D performance in Godot piqued my interest <https://www.reddit.com/r/godot/comments/18oknv5/is_godots_3d_performance_always_this_bad_or_was/> . Most of the answers were actually regarding the graphics performance, and how Godot 3 is much worse than Godot 4 in that regard. None of the answers had any definitive conclusions, so I decided to take a look myself. It’s strongly recommended to check the post and the comments before reading the article.

Disclaimer[s]:

1 — All of the info was gathered solely for the educational purposes. I am not promoting data mining or reverse engineering of any kind.

2 — This post has nothing to do with Wrought Flesh specifically. Developing and releasing a game takes a lot of dedication, time and skill. I fully respect the work behind this game and have no intention of mocking any of the project decisions. The whole purpose of this little profiling project was to check for potential pitfalls, and see what could be done differently in the future to mitigate them.

3 — I’ve been following Godot since 2.x days, but never shipped a commercial / public project with it. I helped shipping quite a few titles with one other famous engine, mobile being my main point of interest. Recently I’ve been working with desktop graphics programming (see why the original post got me interested?).

3 — This post is about Godot 3.x. We need a different game with similar problems, using Godot 4.x, to check the difference.

4 — If you are from Wrought Flesh team and want this post down (or maybe more info about the profiling I have done) — contact me.

My main development setup is a laptop with i9–11980HK and a mobile RTX 3080. I also checked Wrought Flesh on a potato-ish laptop with i7–8550U and Nvidia MX150. I had a lot of useful insights and decided to actually share.

Spoiler #1: Performance problems of Wrought Flesh are not really about 3D rendering.

Among other participants of the original topic, I was convinced that low framerate had everything to do with 3D graphics, so I started with opening the game in my trusty Nvidia Nsight and taking a few snapshots.

Main menu screen ~2.5 ms frame time. So far so good, at least there is a part of the game where performance is showing.

![main menu](001.webp){:width="800" }

Loading into the game we see a realtime cinematic clip in space, and the frame time decreases even more, to about 1 ms/frame.

![cinematic](002.webp){:width="800" }

However, right after loading into the actual map the frame time increases to 20–22 ms/frame. Which completely ruled out the “uncapped framerate” speculation from the comments in the original post.

![main game](003.webp){:width="800" }

So I captured and checked a few frames and found … nothing.

![main game](004.webp){:width="800" }

The capture made no sense. Every time it was something different. The frame looked the same, everything looked exactly like in the frame before, but every time different things seemed to be causing the lag. One frame it’s a part of the terrain in the shadow map. Another frame it’s a small rectangle on the main scene. Yet another frame shows that it’s a completely different thing in the shadow map. Sometimes it was the final compose / postprocessing step of the frame. I once captured a frame where `glClear(GL_DEPTH_BUFFER_BIT)` was apparently taking 15 milliseconds.

Seemingly, when the frame rendering just flies by, Nvidia Nsight gets carried away by all of the software noise on the PC. I made sure to close all the programs except for the profiler and the game and still got the same results. This could only mean one thing — **we are not GPU-bound**. I mean sure, there are a ton of OpenGL warnings about 100% of fragments failing depth test (wasting GPU time due to not having occlusion culling), state change calls that didn’t really change any state (pure driver overhead), the shadow maps being rendered at 4096x4096 at any level of quality. But there was nothing to take 20 milliseconds of render time. I had similar experience with another project that was CPU-bound, but had much busier frames.

This is where things started to get really interesting. I checked the game in Nvidia Systems — a system-wide profiler that can give initial insights on what actually can be causing the lags. The only reason I didn’t start with it is that I was convinced by the original post and comments that it’s really the 3D graphics issue. Also the original author of the post said that their GPU was sitting at 100%, and mine was at 22%–26%.

Anyways, the Systems showed that the process of Wrought Flesh was always utilizing 100% of the CPU available, even though the Task Manager said that the CPU was sitting at 15% tops. Even when I checked load-by-core, it peaked a few times at ~40% on one core.

![main game](005.webp){:width="800" }
![main game](006.webp){:width="800" }

So maybe there’s something that just takes a lot of CPU time and it’s CPU-bound. I was still not convinced because CPU wasn’t trying to boost even to its base speed of 3.3 GHz, sitting at 2.4–2.6 GHz. Another possibility was still driver overhead (Godot 3.x uses OpenGL)

I launched the game on my other laptop with ~100% worse single-core performance CPU and ~1000% worse GPU (quick userbenchmark Intel-to-Intel and Nvidia-to-Nvidia comparison).

**And you know what? It was running at the same 20–25 ms/frame time.**

With CPU being loaded at 32% overall and ~70% load peaks. GPU was also chilling at around 33%. I made a quick check using Nvidia Systems on that laptop and the results were the same — 100% CPU utilization on a per-core basis (with low effective load), low SM / GR utilization on the GPU. One of the commenters of the on the original post noted, that they had “ok” performance with 970m. Well, now we know that MX150 works too.

![main game](007.webp){:width="800" }
![main game](008.webp){:width="800" }

What the hell.

* This is not an uncapped framerate issue. Game runs at 20–25 ms / frame on both of my laptops.
* We are not GPU-bound. Both RTX 3080 and MX150 get barely loaded.
* We are somewhat CPU-bound. 100% CPU utilization, with system reported ~35% effective load.
* There was some driver overhead with OpenGL, but it didn’t seem too severe.

The only remaining reason I have is memory latency and cache efficiency. Both CPUs have comparable memory latency. Other than that, the hardware is really different.

Now I should probably write about my first experience of restoring the original Godot project from a shipped build (Spoiler alert #2 — it’s just a few clicks), how I tried to profile the scripts and found the reasons for the hiccups and hitching, how I changed the source to not include the heaviest part of the scripts. And why the game has a separate “hair quality” setting, even though we barely see any hair.

**But it didn’t really make a difference.**

Even though I managed to shave off a few milliseconds, decrease the severity of the hitching when quickly turning around, frame still took more than 16.6 ms. This is about where I ran out of sensible amount of time to get the-one-definitive answer.

I then quickly went back to the rendering problem, tweaked camera’s far plane to induce more aggressive frustum culling, and the frame still took around ~15 ms. It should have changed a lot in terms of driver overhead — the far plane was at 500 units, and I moved it to 50. But it didn’t really change anything. So I still strongly believe it’s not the driver overhead.

![main game](009.webp){:width="800" }
_Barely rendering anything_

hat I can say with a level of certainty is that the amount of concurrently active scripts on a scene, as well as total number of objects, seems to matter and at least partially causing the apparent memory latency issues.

I tried searching for tools to profile and measure memory latency and cache hit efficiency on Windows to confirm my theory, but I didn’t find anything that I could actually run and use. If you know something about things that work, please let me know.

Still, I consider the information useful:

1 — There’s nothing wrong with rendering of Wrought Flesh or Godot 3.x in general, at least on PC. Both RTX 3080 and a potato MX150 chugs through all of the triangles like it’s nothing. I checked the official platformer example, and it uses the same approach to level design — everything is available at start, everything is rendered, nothing is occlusion-culled etc. It’s just that the level is much smaller and this approach doesn’t seem to scale out of the box, especially if there are a lot of different active objects and a lot of different scripts.

2 — Potentially poor CPU optimization choices only results in occasional hiccups. They were taking 15–30 extra ms per frame on i9–11980HK, and 60–80 ms on another laptop. This is when the CPUs were actually working and the difference in their performance mattered.

I tried upgrading Wrought Flesh to 4.x to make a test build, but it just kept crashing, corrupting the files, permanently staying in the “no response” state. So I decided to leave that for good.

I still think I have more questions than answers. Is it GDScript issue? Is it Godot in general? If so, how different is Godot 4.x from 3.x in terms of memory latency and cache efficiency?
We definitely need more tests, and a working memory latency — cache efficiency profiler.

But the conclusion for now, to answer the original question:

> Is Godot’s 3D Performance Always This Bad?

No, Godot’s 3D performance for PC is actually OK. There is something in the engine that seems to make things slow, but almost certainly it’s not the graphics.