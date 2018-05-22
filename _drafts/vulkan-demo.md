---
layout: post
title: "Writing a demo in Vulkan"
categories: blog
---
I recently coded a new [demoscene](https://en.wikipedia.org/wiki/Demoscene)
production, called [Aurora](http://www.pouet.net/prod.php?which=75791). Unlike
most of my recent-ish productions, this one stands out in that it was written
using [Vulkan](https://www.khronos.org/vulkan/), a high-performance, explicit
API. This post will explain a bit about why, what worked, and what could have
been done better.

## Background

Since around 2007, most of my demoscene productions has been written using my
[Very Last Engine Ever](https://github.com/excess-demogroup/vlee) framework
(VLEE for short). VLEE was written out of frustration with the complexity of
writing reusable, high-level engines, and was less of an engine and more of a
collection of utilities that made open-coding demos easier. VLEE was written
using [DirectX 9](https://en.wikipedia.org/wiki/Direct3D#Direct3D_9), which
was the latest version of DirectX with a reasonable amount of hardware support
at the time. Fast forward 10 years, and I've still been using the same engine,
and because DirectX 9 isn't forwards-compatible, I've been limited to
DX9-level functionality.

I've tried to port to DirectX 10/11 several times during those 10 years, but
neither of those APIs rubbed me the right way. There were always some detail
that just felt awful. So I've stayed behind instead so far. :grimacing:

When Vulkan came out a few years ago, I decided to give it a try. I'm a
rendering-guy who have written GPU-drivers pretty much from scratch in the
past, so explicit APIs don't really scare me.

## Early hacks

For a very long time, the new engine I wrote wasn't very impressive. Basically
just a rotating, textured cube. Visually, nothing exciting at all. But most
graphics coders know that this is kinda the point where you can start doing
just about anything.

The big blocker for a really long time, ended up being post-processing. I
insisted on doing my post-processing in a compute-shader, writing directly to
the swapchain image. While that's possible on some GPUs, and I eventually got
it working, it turns out that's not as good of an idea as I though.

Anyway, whenever I tried to output from the compute-shader to the swapchain
image, all I got was black. No validation-errors or nothing. Just black.
Nothing I tried worked. I was stuck. :dizzy_face:

## Breakthrough

I wasn't planning on doing a demo for Revision 2018. But about three weeks
before the party, I had a breakthrough. I found some other code out there that
output directly to the swapchain from the compute-shader, and it worked on my
machine. So what I did was reduce my program until it was doing the same work
as the other program, until I finally figured out what the difference was;
sRGB writes. :relieved:

Yeah, so the compute-shader can't do sRGB writes, I needed to do linear -> sRGB
in the shader instead. Sadly, the Vulkan Validation Layers hadn't picked that
up. :disappointed: That would be a nice change to the validation layer, IMO. Maybe one day
when I have some spare time...

After getting it to work, I quickly changed to doing an explicit blit from a
temp image to the swapchain image for portability-reasons anyway. Turns out,
intel GPUs don't seem to support it.

## Main effect

OK, so after a bit of refactoring and playing around, I got an effect that I'd
been wanting to implement for quite some time; the old-school vector-slime
effect, on new-school hardware.

The basic idea is pretty simple:
- Each frame, we render the scene at the current time-stamp.
- We keep track of the last n framebuffers (in this case 128, or a bit more
  than two seconds of frames at 60hz).
- Using some screen-position to time-offset mapping, we sample the
  corresponding pixel with the time-offset. In our case, that's using an
  offset-map (a texture).

In addition, I decided to play around with motion-blur with "chromatic
aberration". That is, I sample each pixel a bunch of times, over some range,
filtering the time over the RGB-spectrum.

I needed some content to apply this post-processing effect on, though. So I
ported an old shader that I never ended up using in a demo before. It
raytraces a refracted plane inside a 3D object, approximating chromatic
aberration by sampling a texture on that plane in a line, convoluted with the
RGB-spectrum. Yeah, it's a very typical me-thing to do! :innocent:

When I got all of this working, about two weeks before the party, I kinda
felt that I had *something*. And my girlfriend was working on a demo, which
made me a bit jealous of her fun! :stuck_out_tongue_winking_eye: So, I poked
[Gloom](https://twitter.com/gloom303), my usual partner in crome, and he was
game! We decided to make a demo out of it.

## Last minute hacks

So, this demo was written in a hurry, using an engine that doesn't really do
a lot. This means that some pretty nasty short-cuts were made.

I early on decided to not even try to do memory-allocations properly. I
already knew that I wouldn't have to do a lot of unrelated textures and
buffers. Instead, I opted on saving everything that easily batched together
into the same allocation. No clever memory allocator that can bite me last
minutes. This turned out nicely.

What didn't turn out so nicely, was that I didn't do any shader-program
abstraction at all. This was a major pain, because every time I added a new
shader-program, I had to write a bunch of descriptor layouts, and manual
descriptor-updates. Adding new shader-programs were a major pain, but there
were no time for a rewrite. :watch: So I pushed back as much as I could on Gloom's
shader-requests. But to keep things interesting, we needed *some* variation,
and I was unable to keep it at zero.

Let's just say that I'm not going to make this mistake for the next demo. In
fact, I've written an abstraction that seems to work.

### Sync + Feedback effects = Hell

I guess this goes without saying, but doing any kind of syncing together with
feedback-effects is kinda nightmarish. Gloom struggled with this a lot, but
with some tweaks here and there, he managed to get things working. I have no
idea how, but it somehow worked out.

### Color grading

To make things a bit more visually interesting, I quick-hacked some
data-driven color-grading. I added support for loading LUT-files (exported
from Adobe Photoshop) into 3D textures, and weighing a couple of them to
tweak the colors.

This did a lot for the looks, and allowed us to easily separate the different
effects a bit visually.

There's more color-grading we would have preferred to do, but we ran out of
time...

## Finishing up

This time I ended up pretty much coding all the way through the party, with
the exception of the last day, which mostly consisted of Gloom
[syncing](https://rocket.github.io/) the demo, and me handing it in and
testing on the compo-machine.

Sadly, this being a new engine and all, the handing in-part was anything but
smooth. I feed bad for Chaos for having to deal with me way too many times,
coming with way too many new builds. In the end, we got a version that worked
fine.

## Afterthoughts

I kind of expected something to feel wrong along the way, throwing me back to
the good, old DX9 world. But that just never really happened this time. Maybe
the pile of ideas about things I could do with modern GPU-features made me
solider through?

Anyway, I think it was a great learning experience for me. It's my first time
playing with Vulkan, and I'm definitely staying with Vulkan for now!
