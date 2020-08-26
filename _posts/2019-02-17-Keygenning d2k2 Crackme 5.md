---
layout: post
published: true
title: Keygenning diablo2oo2's crackme 5
---
## Overview

The following tutorial will document how to unpack and program a keygenerator for diablo2oo2's
fifth crackme.

Experience in x86 assembly is assumed.

## Unpacking the crackme.

Load the crackme into x32dbg. Then, navigate to the "Memory Map" tab:

![d2k1.PNG]({{site.baseurl}}/images/crackme5/d2k1.PNG)

The executable has 2 PE sections, each with the name "d2k2".
Place a one-shot on-execution breakpoint on the upper "d2k2" section,
with the RWC permissions.

![d2k2.png]({{site.baseurl}}/images/crackme5/d2k2.png)

Run the crackme until it breaks on the breakpoint set.

![d2k3.png]({{site.baseurl}}/images/crackme5/d2k3.png)

Then, click on the "Scylla" icon.

![d2k4.png]({{site.baseurl}}/images/crackme5/d2k4.png)

Click on the IAT Autosearch, then click "Get Imports.

![d2k5.png]({{site.baseurl}}/images/crackme5/d2k5.png)

Click "Dump" to save a unpacked copy of the executable.
Then, click "Fix Dump" to rebuild the executable into a working one.
This will be the one to use when debugging.
Unload this executable from x32dbg.


## Reverse engineering the crackme.

Load up the freshly depacked executable in x32dbg again. 
The debugger should load the executable at the following code at the executable's real entrypoint, since the packer's loader is now removed.

![d2k6.PNG]({{site.baseurl}}/images/crackme5/d2k6.PNG)

Run the crackme in x32dbg.

From here, it is needed to place any possible breakpoints on places which the user enters any
data, say for instance username and serial key. 

![d2k7.png]({{site.baseurl}}/images/crackme5/d2k7.png)

![d2k8.png]({{site.baseurl}}/images/crackme5/d2k8.png)

Stepping through our entered serial number and username, it becomes immediately apparent that the length of the serial key is checked and tested against the length entered. In this case our entered serial, which is 5 characters in length does not match the limit, which is 0x20 characters.
In this case the check will fail and jump over a vast length of code to drop at the following....

![d2k9.png]({{site.baseurl}}/images/crackme5/d2k9.png)

Which is precisely where we don't want to be. We want to pass the check, **so our serial key must be exactly 0x20 (in hex) characters in length**

So we rerun the check in the debugger, only this time trying a serial key which is precisely 32 characters (0x20 in hex) in length. Doing so gets us to the following:

![d2k10.png]({{site.baseurl}}/images/crackme5/d2k10.png)

This check checks the username for the length of characters. It must be greater than 4 characters in length and less than 32 characters (20 in hex).

From here if the name passes those checks it moves to the first portion of the serial algorithm.
For the sake of brevity the following screen illustrates how this portion of the serial check works:

![d2k11.png]({{site.baseurl}}/images/crackme5/d2k11.png)

Once that routine is complete, the check moves to a much longer and more complex loop....

![d2k12.png]({{site.baseurl}}/images/crackme5/d2k12.png)

Once again, the above picture is used for brevity. One very worthy point of note is the special call which is called multiple times in the main serial loop:

![d2k13.png]({{site.baseurl}}/images/crackme5/d2k13.png)

This call is full of binary math, constants and very lengthy and outputs a 128bit length bytestream. One thing of note is that the constants are based off the MD5 hash algorithm, same with some of the logic seems derived from it, however there has been many modifications such as "round" constants being tweaked.

Once the subcall has been run, it exits back to the main logic of the serial routine.

![d2k14.PNG]({{site.baseurl}}/images/crackme5/d2k14.PNG)

The rest of the hashing loop is documented here.

Once the loop is complete, the next phase is undertaken.

![d2k15.PNG]({{site.baseurl}}/images/crackme5/d2k15.PNG)

The formatting of the hash output is then done and concatenated into one large 32 character string.

The final string is then checked one final time, one character at a time using some byte level arithmetic and division. This is where the actual serial check takes place. If all the per-character checks pass, then the crackme is complete.

![d2k16.PNG]({{site.baseurl}}/images/crackme5/d2k16.PNG)

## Writing the keygen.

Writing the keygen involved some debugging to check that the operations are perfect, byte per byte. The 128bit hash function was not reversed, however it was code spliced from the original crackme using [fearless' CopyToAsm plugin for x64dbg.](https://github.com/mrfearless/CopyToAsm-Plugin-x86)

[Source code to the keygen is here.](https://github.com/mudlord/crackme_solutions/blob/master/keygenned/algo/d2k2_crackme05.c)