---
layout: post
published: true
title: WTFweg retrospective
---

## Overview

For me, I quite like the libretro concept: modular components with a framework binding it all, allowing for cross platform source ports of games, emulators, media applications, etc. For several  years I would work within the reference frontend to work on such things namely RetroArch. For many difference reasons to me such as:

- Bloat/scope creep
- An absolute clinging to old tech regardless of its date, thus introducing more scope creep
- Confusing interface

I decided to work on my own implementation of the libretro API, since RetroArch to me was too clumsy in many different ways to work with just to debug cores, while keeping as minimum scope creep as possible by limiting to the hardware I actually use. This is a retrospective on my personal means to mitigate these, to me, problems.

## einweggerat

The first major step for me was to write a Windows only libretro core loader, since at the time  I was using Windows most. Three major things popped out of me:

- The need to debug cores set from the commandline.
- A small enough codebase needed to test cores.
- Source level debugging to debug cores as they are ran.

Thus [einweggerat](https://github.com/mudl0rd/einweggerat) was born. It was purely a means to run cores with the bare minimum with full OpenGL core support to debug the things I was working on at the time, namely the N64 cores in the libretro ecosystem. It however had a major caveat: to debug MSYS2 compiled cores under Microsoft Visual Studio, conversion of debug symbols with the usage of an outside utility was needed. This to me was very clunky and thus it was scrapped. However it did meet my goals for what I wanted to do at the time.

I eventually got as far as into adding a WTL based interface for einweggerat thus allowing for easy core loading, configuring joypad/game inputs per core by easy to see descriptions, and not by the clunky "RetroPad" interface RetroArch enforces, as well as an easy interface to change settings. 

![1.png]({{site.baseurl}}/images/wtfweg/d2t7WD9.png)
![1.png]({{site.baseurl}}/images/wtfweg/uZcxffp.png)

But to me, the interface was holding me back as well as the need to test Linux cores too. Thus WTFweg was made, based on residual code from einweggerat.

## WTFweg

WTFweg is my current latest attempt to do what I need or wanted, or what I wanted in a RetroArch-less means to run libretro stuff, at all.

Over time to it I added:

* [Dynamic rate control](https://docs.libretro.com/development/cores/dynamic-rate-control/).
* Support for GL-based libretro cores.
* Support for contentless libretro cores (like game engine reimplementations)
* Support for transparently loading Windows libretro cores from ZIP/RAR/7z files.
* Savestates/SRAM saving/loading.
* Commandline support for debugging cores with GDB.
* Per-game settings per-core.
* Multiple controller support.

The major hurdle of debugging cores and converting symbols was completely sidestepped by rewriting einweggerat's interface, and converting to a workflow of using MSYS2 (hence GCC), SDL, Meson, and Visual Studio Code. All code was replaced with C++17. All code also compiles on modern Linux, mainly tested with Linux Mint.

Most of the user interface hurdles are bypassed through [dear-imgui](https://github.com/ocornut/imgui), for relevant user interface duties, rather than Qt or some other UI toolkit.

![1.png]({{site.baseurl}}/images/wtfweg/1.png)
![1.png]({{site.baseurl}}/images/wtfweg/2.png)
![1.png]({{site.baseurl}}/images/wtfweg/3.png)
![1.png]({{site.baseurl}}/images/wtfweg/4.png)
![1.png]({{site.baseurl}}/images/wtfweg/5.png)

 There is a flaw however to me that the renderer for dear-imgui is bound to the current one used to render core visuals. So if a Vulkan core is used, a Vulkan UI renderer has to be implemented and due to WTFweg's design, the renderer for all has to be reintialized per core to match. This could be mitigated with asset sharing between OpenGL 4.6 and Vulkan, however, using relevant GL extensions, with zero performance hit. Of course, on certain platforms like phones or Raspberry Pi, plain GLES2 can be mandated.

Transparently loading Windows libretro cores was a major goal for me, since there can be some tedium alleviated from extracting cores manually for use in WTFweg when downloaded from the buildbot. This is done using a custom PE DLL loading wrapper which manually maps DLLs from memory, including all exports. This includes TLS callback emulation, exception directory emulation, etc. This could also eventually allow for downloading cores directly from the buildbot and using on a game by game basis with the DLL (or download ZIP, even) being on disk. Theorectically, this can also be done for Linux, with manually mapping ELF objects in a similar process being done for PE libraries.

Having an interface that expanded on einweggerat was vitally important for me, since I wanted to explore debugging other cores down the line. This included input selection for various devices including mice input, etc.

![1.png]({{site.baseurl}}/images/wtfweg/9.png)

"RetroPad", the abstraction libretro and RetroArch uses, is still used as a fallback on cores that rely so, with means to configuring it when a core isn't loaded, or such a core uses the "RetroPad" input abstraction. One of the major design goals of WTFweg was to remove it as much out of sight, out of mind as possible.

![1.png]({{site.baseurl}}/images/wtfweg/7.png)
![1.png]({{site.baseurl}}/images/wtfweg/8.png)

There is still much for me to work personally on WTFweg such as:

* MIDI output support in a cross platform manner.
* Vulkan core support.
* [Postprocess slang shader support](https://github.com/libretro/slang-shaders).
* Postprocess audio filter support.
* Bugfixes for the remaining cores untested.
* Internal CD/DVD mounting support for cores using the VFS CD interface.