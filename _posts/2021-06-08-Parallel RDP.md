---
layout: post
published: true
title: paraLLEl-RDP ported to PJ64 and Mupen64Plus
---
# tldr;

I have ported Themaister's [paraLLEl-RDP](https://github.com/Themaister/parallel-rdp) to Project64 and Mupen64Plus.
It is most usable in loganmc10's ["m64p"](https://github.com/loganmc10/m64p) project. 
Do not expect support/help on my end. I originally did this entirely for my own personal use.
Use at your own risk. 

## Overview

The N64 scene for some time has often for RDP emulation resorted to plugins (addons) for
N64 emulators to perform the tasks needed for graphics emulation. These often used a 
high level emulation approach, and with varying middling degrees of success, a low level.

Various different APIs were, up to this point in time, used such as:
* Glide3x/Glide2x
* OpenGL (up to version 4.3)
* DirectX (various editions, up to version 9.0)


There is an low-level software rasterizing RDP emulator plugin by the name of [Angrylion-RDP](https://github.com/project64/angrylion-rdp). Due to the lack of vectorization as well as threading (though a [multithreading fork recently exists](https://github.com/ata4/angrylion-rdp-plus), but to me has its own set of problems), this has been somewhat very prohibitive to use apart for low level debugging N64 RDP behaviour/accuracy testing

Recently, technology has moved to a point where it is feasible to pixel-level emulate the output of the Nintendo 64, by emulating the RDP's behaviour precisely, completely on the GPU. This could be done by porting the logic of "Angrylion-RDP" entirely as compute shaders. This was the genesis behind Themaister (a developer previously working for ARM's Mali division, now Valve) making a dot-accurate emulator using Vulkan. The current version of paraLLEl-RDP is the latest iteration of this work and is extremely accurate for RDP simulation, It was originally designed for the N64 libretro core "paraLLEl", but now it has been made modular to the point various other emulators now use it too for RDP emulation, such as Near's [Ares](https://ares.dev/) and dillonb's ["N64"](https://github.com/Dillonb/n64).

For some time, I wanted to use this RDP emulator in other mainstream emulators, since there is major benefits to its use, such as being extremely fast (on my hardware) as well as being pixel-level accurate. I, for some time, always wanted a easy way to use Themaister's work in another emulator, such as Project64 and Mupen64Plus (since for various reasons I personally somewhat dislike using RetroArch, to the point of writing my own libretro-spec emulator core loader exclusively for Windows and N64 emulation, with somewhat decent success). Figuring that Mupen64Plus already works to a degree with most commercial games, as well as Project64 (using cxd4's RSP plugin, since zilmar's RSP plugin has various accuracy problems that cause problems with LLE graphics emulation), I figured I do it myself since to me no one else for some reason will, even if initially it was for my own personal benefit.

My personal system for the port and testing is as follows:
![I-Have-No-Idea-What-Im-Doing-1.jpg]({{site.baseurl}}/images/pcspecs.PNG)


## Process

A lot of the porting process came from reading "Parallel"'s (the libretro core) code to somewhat componetize it into a standalone Mupen64Plus plugin, slowly debugging on the way any problems when making a GNU makefile to build a DLL and testing with the standalone version of Mupen64Plus, in MSYS2. Since internally Parallel uses Mupen64Plus plugins statically linked, this was rather trivial to replicate the interface. Using the standalone Git repository of paraLLEl-RDP, the basic structure was replicated, leaving some sifting through paraLLEl's makefile system to determine how paraLLEl-RDP's source is built. However, I figured since there would need to be significant upstream changes to Mupen64Plus's core and frontend executable to support Vulkan natively, I'd rather implement in a different way.

I would use a OpenGL rendering context (since I know OpenGL, from my past work, fairly well) to blit the contents of a image buffer that parallE1-RDP creates, since I was very unfamiliar with Vulkan on a graphics programmer level. This process was originally done for Near's Ares emulator (probably for very similar reasons), in that Themaister created a function call to copy VI buffer contents asynchronously to a C++ vector. I made a similar function, however it handles the copies synchronously, with support for various VI options to disable things such as dithering.

The OpenGL blitting code itself is mostly salvaged from ata4's Angrylion-RDP fork, since to me it worked rather well, and to be honest, felt at the time didn't like reinventing the wheel since to me at the time, for my own personal use, I didn't see the harm.

The Project64 port, was also very trivial, due to the vast similarities between Mupen64Plus's plugin API and PJ64's. During personal bugtesting, a process of compiling in GCC and then converting the DLL's debug symbols to PDB format was used to use MSVC as a debugger, due to its superior (to me) source level debugging abilities. 

## Problems

As far as I know there is still some bugs from my ports, which I have been doing entirely in my spare time and whenever I feel motivated to program, rather than forcing myself time to do so. Other people, namely [Rosalie241](https://github.com/Rosalie241) and [Logan](https://github.com/loganmc10) have vastly fixed issues present as well as rewriting image copying to be more performant.

As far as I know:

- On Project64, there is no exclusive fullscreen.
- On both ports, there is no plain Vulkan texture blitting and only one rendering context, everything is displayed on output using OpenGL 3.3, and computed using a headless Vulkan instance. 
- On PJ64's side, various accuracy problems or downright freezes can occur when using Zilmar's RSP plugin. This is due to various RSP vector opcodes not being implemented correctly. As a workaround, when using Project64, using cxd4's RSP plugin can help with that, although that can some with a speed penalty on some systems.
- 8x image rescaling will not work on the vast majority of video cards, due to the sheer amount of compute horsepower needed at that level. Its known to work on Nvidia RTX 3090's and similar level hardware. 4x image rescaling though, yields major benefits as well as working on systems that can handle it.

## Source code

[PJ64 port.](https://github.com/mudlord/pj64-parallelrdp)

[Mupen64Plus port.](https://github.com/mudlord/mupen64plus-video-parallel)