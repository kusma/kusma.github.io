---
layout: post
title: "Introducing Zink: OpenGL on Vulkan"
categories: blog
canonical_url: "https://www.collabora.com/news-and-blog/blog/2018/10/31/introducing-zink-opengl-implementation-vulkan/"
---
For the last month or so, I've been playing with [a new project](https://gitlab.freedesktop.org/kusma/mesa/tree/zink)
during my work at [Collabora](https://www.collabora.com/), and as I've
[already briefly talked about](https://www.youtube.com/watch?v=ukrB-Lbl_Jg) at
[XDC 2018](https://xdc2018.x.org/), it's about time to talk about it to a
wider audience.

## What is Zink?

Zink is an [OpenGL](https://www.opengl.org/) implementation on top of
[Vulkan](https://www.khronos.org/vulkan/). Or to be a bit more specific, Zink
is a [Mesa](https://www.mesa3d.org/) Gallium driver that leverage the existing
OpenGL implementation in Mesa to provide hardware accelerated OpenGL when only
a Vulkan driver is available.

<figure>
  <img src="{{site.url}}/assets/zink-gears.png" alt="glxgears on Zink">
  <figcaption>glxgears on Zink</figcaption>
</figure>

Here's an overview of how this fits into the Mesa architecture, for those unfamiliar with it:

<figure>
  <div>
    <div style="border-radius: 4px; text-align: center; overflow: hidden; text-overflow: ellipsis; font-size: 1.25rem; color: white; background: #5c3dcc;">Application</div>
    <center><svg height="1em" viewbox="-5 -5 30 30"><path style="stroke: currentColor; stroke-width: 5; stroke-linecap: round;" d="M0 10l10 10M20 10l-10 10M10 0v20"/></svg></center>
    <div style="border-radius: 4px; text-align: center; overflow: hidden; text-overflow: ellipsis; font-size: 1.25rem; color: white; background: #5c3dcc; padding: 5px">Mesa
      <div style="border-radius: 4px; text-align: center; overflow: hidden; text-overflow: ellipsis; font-size: 1.25rem; color: white; background: #5c3dcc; box-shadow: inset 0px 0px 0px 2px #43c200;">Gallium OpenGL State Tracker</div>
      <div style="font-size: 1rem; width: 50%; padding-right: 2.5px; float: left;">
        <center><svg height="1em" viewbox="-5 -5 30 30"><path style="stroke: currentColor; stroke-width: 5; stroke-linecap: round;" d="M0 10l10 10M20 10l-10 10M10 0v20"/></svg></center>
        <div style="border-radius: 4px; text-align: center; overflow: hidden; text-overflow: ellipsis; font-size: 1.25rem; background: #43c200; color: black">Zink</div>
      </div>
      <div style="font-size: 1rem; width: 50%; padding-left: 2.5px; float: left;">
        <center><svg height="1em" viewbox="-5 -5 30 30"><path style="stroke: currentColor; stroke-width: 5; stroke-linecap: round;" d="M0 10l10 10M20 10l-10 10M10 0v20"/></svg></center>
        <div style="border-radius: 4px; text-align: center; overflow: hidden; text-overflow: ellipsis; font-size: 1.25rem; color: white; background: #5c3dcc; box-shadow: inset 0px 0px 0px 2px #43c200;">Other Gallium drivers</div>
      </div>
    </div>
    <div style="width: 50%">
    <center><svg height="1em" viewbox="-5 -5 30 30"><path style="stroke: currentColor; stroke-width: 5; stroke-linecap: round;" d="M0 10l10 10M20 10l-10 10M10 0v20"/></svg></center>
    <div style="border-radius: 4px; text-align: center; overflow: hidden; text-overflow: ellipsis; font-size: 1.25rem; color: white; background: #5c3dcc;">Vulkan</div>
    </div>
  </div>
  <figcaption>Architectural overview</figcaption>
</figure>

## Why implement OpenGL on top of Vulkan?

There's several motivation behind this project, but let's list a few:

1. Simplify the graphics stack
2. Lessen the work-load for future GPU drivers
3. Enable more integration
4. Support application porting to Vulkan

I'll go through each of these points in more detail below.

But there's another, less concrete reason; **someone** had to do this. I was
waiting for someone else to do it before me, but nobody seemed to actually go
ahead. At least as long as you don't count solutions who only implement some
variation of OpenGL ES (which in my opinion doesn't solve the problem; we need
**full** OpenGL for this to be *really* valuable).

### 1. Simplifying the graphics stack

One problem is that OpenGL is a **big** API with a **lot** of legacy stuff
that has accumulated since its initial release in 1992. OpenGL is
well-established as a requirement for applications and desktop compositors.

But since the very successful release of Vulkan, we now have two main-stream
APIs for essentially the same hardware functionality.

It's not looking like neither OpenGL nor Vulkan is going away, and the
software-world is now hard at work implementing Vulkan support everywhere,
which is **great**. But *this leads to complexity*. So my hope is that we can
simplify things here, by only require things like desktop compositors to
support *one* API down the road. We're not there yet, though; not all hardware
has a Vulkan-driver, and some older hardware can't even support it. But at
some point in the not too far future, we'll probably get there.

This means there might be a future where OpenGL's role *could* purely be one
of legacy application compatibility. Perhaps Zink can help making that future
a bit closer?

### 2. Lessen the work-load for future GPU drivers

The amount of drivers to maintain is only growing, and we want the amount of
code to maintain for legacy hardware to be as little as possible. And since
Vulkan is a requirement already, maybe we can get *good enough* performance
through emulation?

Besides, in the Open Source world, there's even new drivers being written for
old hardware, and if the hardware is capable of supporting Vulkan, it *could*
make sense to only support Vulkan "natively", and do OpenGL through Zink.

It all comes down to the economics here. There aren't infinite programmers
out there that can maintain every GPU driver forever. But if we can make it
easier and cheaper, maybe we can get better driver-support in the long run?

### 3. Enable more integration

Because Zink is implemented as a Gallium driver in Mesa, there's some
interesting side-benefits that comes "for free". For instance, projects like
Gallium Nine or Clover could *in theory* work on top of the i965 Vulkan driver
through Zink. Please note that this hasn't really been tested, though.

It should also be possible to run Zink on top of a closed-source Vulkan driver,
and still get proper window system integration. Not that I promote the idea of
using a closed-source Vulkan driver.

### 4. Support application porting to Vulkan

This might sound a bit strange, but it might be possible to extend Zink in
ways where it can act as a cooperation-layer between OpenGL and Vulkan code in
the same application.

The thing is, big CAD applications etc won't realistically rewrite all of
their rendering-code to Vulkan in a wave of a hand. So if they can for instance
prototype some Vulkan-code inside an OpenGL application, it might be easier to
figure out if Vulkan is worth it or not for them.

## What does Zink require?

Zink currently requires a Vulkan 1.0 implementation, with the following
extensions (there's a few more, due to extensions requiring other extensions,
but I've decided to omit those for simplicity):

- `VK_KHR_maintenance1`: This is required for the viewport flipping. It's also
   possible to do without this extension, and we have some experimental
   patches for that. I would certainly love to require as few extensions as
   possible.
- `VK_KHR_external_memory_fd`: This is required as a way of getting the
   rendered result on screen. This isn't *technically* a hard requirement, as
   we also have a copy-based approach, but that's almost unusably slow. And
   I'm not sure if we'll bother keeping it around.

Zink has to my knowledge *only* been tested on Linux. I don't think there's
any major reasons why it wouldn't run on any other operating system supporting
Vulkan, apart from the fact that some window-system integration code might
have to be written.

## What does Zink support?

Right now, it's not super-impressive: we implement **OpenGL 2.1**, and **OpenGL
ES 1.1 and 2.0** plus some extensions. Please note that the list of extensions
might depend on the Vulkan implementation backing this, as we forward
capabilities from that.

The list of extensions is too long to include here in a sane way, but [here's
a link](https://gitlab.freedesktop.org/kusma/mesa/snippets/518/raw) to the
output of glxinfo as of today on top of i965.

Here's some screenshots of applications and games we've tested that renders
more or less correctly:

<figure>
  <img src="{{site.url}}/assets/zink-openarena.png" alt="OpenArena on Zink">
  <figcaption>OpenArena on Zink</figcaption>
</figure>

<figure>
  <img src="{{site.url}}/assets/zink-weston.png" alt="Weston on Zink">
  <figcaption>Weston on Zink</figcaption>
</figure>

<figure>
  <img src="{{site.url}}/assets/zink-quake3.png" alt="Quake 3 on Zink">
  <figcaption>Quake 3 on Zink</figcaption>
</figure>

<figure>
  <img src="{{site.url}}/assets/zink-etr.png" alt="Extreme Tux Racer on Zink">
  <figcaption>Extreme Tux Racer on Zink</figcaption>
</figure>

## What doesn't work?

Yeah, so when I say OpenGL 2.1, I'm ignoring some features that we simply do
not support yet:

- `glPointSize()` is currently not supported. Writing to `gl_PointSize` from
  the vertex shader *does* work. We need to write some code to plumb this
  through the vertex shader to make it work.
- Texture borders are currently always black. This will also need some
  emulation code, due to Vulkan's lack of arbitrary border-color support.
  Since a lot of hardware actually support this, perhaps we can introduce some
  extension to add it back to the API?
- No control-flow is supported in the shaders at the moment. This is just
  because of lacking implementation for those opcodes. It's coming.
- No `GL_ALPHA_TEST` support yet. There's some support code in NIR for this,
  we just need to start using it. This will depend on control-flow, though.
- `glShadeModel(GL_FLAT)` isn't supported yet. This isn't particularly hard or
  anything, but we currently emit the SPIR-V before knowing the drawing-state.
  We should probably change this. Another alternative is to patch in a
  flat-decoration on the fly.
- Different settings for `glPolygonMode(GL_FRONT, ...)` and
  `glPolygonMode(GL_BACK, ...)`. This one is *tricky* to do correct, at least
  if we want to support newer shader-stages like geometry and tessellation at
  the same time. It's also *hard* to do performant, even without these
  shader-stages, as we need to draw these primitives in the same order as they
  were specified but with different primitive types. Luckily, Vulkan can do
  pretty fast geometry submission, so there might be some hope for some
  compromise-solution, at least. It might also be possible to combine
  stream-out and a geometry-shader or something here if we really end up
  caring about this use-case.

And most importantly, we are *not* a conformant OpenGL implementation. I'm not
saying we will never be, but as it currently stands, we do not do conformance
testing, and as such we neither submit conformance results to Khronos.

It's also worth noting that at this point, we tend to care more about
applications than theoretical use-cases and synthetic tests. That of course
doesn't mean we do not care about correctness at all, it just means that we
have plenty of work ahead of us, and the work that gets us most real-world
benefit tends to take precedence. If you think otherwise, please send some
patches! :wink:

## What's the performance-hit compared to a "native" OpenGL driver?

One thing should be very clear; a "native" OpenGL driver will always have a
better performance-potential, simply because anything clever we do, they can
do as well. So I don't expect to beat any serious OpenGL drivers on
performance any time soon.

But the performance loss is already kinda less than I feared, especially since
we haven't done anything particularly fancy with performance yet.

I don't yet have any systematic benchmark-numbers, and we currently have some
kinda stupid bottlenecks that should be very possible to solve. So I'm
reluctant to spend much time on benchmarking until those are fixed. Let's just
say that I can play Quake 3 at tolerable frame rates right now ;)

But OK, I will say this: I currently get around 475 FPS on glxgears on top of
Zink on my system. The i965 driver gives me around 1750 FPS. *Don't read too
much into those results, though*; glxgears isn't a good benchmark. But for
that particular workload, we're about a quarter of the performance. As I said,
I don't think glxgears is a very good benchmark, but it's the only thing
*somewhat* reproducible that I've run so far, so it's the only numbers I have.
I'll certainly be doing some proper benchmarking in the future.

In the end, I suspect that the pipeline-caching is going to be the big hot-spot.
There's a lot of state to hash, and finally compare once a hit has been found.
We have some decent ideas on how to speed it up, but there's probably going
to be some point where we simply can't get it any better.

But even then, perhaps we could introduce some OpenGL extension that allows an
application to "freeze" the render-state into some objects, similar to [Vertex
Array Objects](https://www.khronos.org/opengl/wiki/Vertex_Specification#Vertex_Array_Object),
and that way completely bypass this problem for applications willing to do a
bit of support-code? The future will tell...

All in all, I'm not too worried about this yet. We're still early in the
project, and I don't see any major, impenetrable walls.

## How to use Zink

Zink is only available as source code at the moment. No distro-packages exits
yet.

### Requirements

In order to build Zink, you need the following:
- Git
- Build dependencies to compile Mesa
- Vulkan headers and libraries
- [Meson](http://mesonbuild.com/) and [Ninja](https://ninja-build.org/)

### Building

The code currently lives in [the zink-branch](https://gitlab.freedesktop.org/kusma/mesa/tree/zink)
in my [Mesa fork](https://gitlab.freedesktop.org/kusma/mesa).

The first thing you have to do, is to clone the repository and build the
`zink`-branch. Even though Mesa has an autotools build-system, Zink only
supports the Meson build-system. Remember to enable the `zink` gallium-driver
(`-Dgallium-drivers=zink`) when configuring the build.

Install the driver somewhere appropriate, and use the `$MESA_LOADER_DRIVER_OVERRIDE`
environment variable to force the `zink`-driver. From here you should be able
to run many OpenGL applications using Zink.

Here's a rough recipe:
<pre style="border-radius: 3px; background-color: #111; color: #999;">
$ <span style="color: #fff;">git clone https://gitlab.freedesktop.org/kusma/mesa.git mesa-zink</span>
Cloning into 'mesa-zink'...
<span style="color: #555;">...</span>
Checking out files: 100% (5982/5982), done.
$ <span style="color: #fff;">cd mesa-zink</span>
$ <span style="color: #fff;">git checkout zink</span>
Branch 'zink' set up to track remote branch 'zink' from 'origin'.
Switched to a new branch 'zink'
$ <span style="color: #fff;">meson --prefix=/tmp/zink -Dgallium-drivers=zink build-zink</span>
The Meson build system
<span style="color: #555;">...</span>
Found ninja-X.Y.Z at /usr/bin/ninja
$ <span style="color: #fff;">ninja -C build-zink install</span>
ninja: Entering directory &#x60;build-zink'
<span style="color: #555;">...</span>
installing /home/kusma/temp/mesa-zink/build-zink/src/gallium/targets/dri/libgallium_dri.so to /tmp/zink/lib64/dri/zink_dri.so
$ <span style="color: #fff;">LIBGL_DRIVERS_PATH=/tmp/zink/lib64/dri/ MESA_LOADER_DRIVER_OVERRIDE=zink glxgears -info</span>
GL_RENDERER   = zink (Intel(R) UHD Graphics 620 (Kabylake GT2))
GL_VERSION    = 2.1 Mesa 18.3.0-devel (git-395b12c2d7)
GL_VENDOR     = Collabora Ltd
GL_EXTENSIONS = GL_ARB_multisample GL_EXT_abgr <span style="color: #555;">...</span>
</pre>

### Submitting patches

Currently, the development happens on `#dri-devel` on [Freenode](https://freenode.net/).
Ping me (my handle is `kusma`) with a link your branch, and I'll take a look.

## Where do we go from here?

Well, I think "forwards" is the only way to move :wink:. I'm currently working
1-2 days per week on this at Collabora, so things will keep moving forward on
my end. In addition, Dave Airlie seems to have a high momentum at the moment
also. He has a work-in-progress branch that hints at GL 3.3 being around the
corner!

I also don't think there's any fundamental reason why we shouldn't be able to
get to full OpenGL 4.6 eventually.

Besides the features, I also want to try to get this upstream in Mesa in some
not-too-distant future. I think we're already beyond the point where Zink is
useful.

I also would like to point out that [David Airlie](https://airlied.blogspot.com/)
of [RedHat](https://www.redhat.com/) has contributed a *lot* of great patches,
greatly advancing Zink from what it was before his help! At this point, he has
implemented at least as many features as I have. So this is very much his
accomplishment as well.
