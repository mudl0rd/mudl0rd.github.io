---
layout: post
published: true
title: Keygenning diablo2oo2's crackme 1
---

**Meant to be a a simple tutorial in debugging with x64dbg**

The crackme can be found [here.](https://github.com/mudlord/crackme_solutions/blob/master/crackmes/d2k2_crackme1.zip)

![1.png]({{site.baseurl}}/images/crackme1/d2k2crkme1.PNG)

Load up the crackme in x64dbg.

![1.png]({{site.baseurl}}/images/crackme1/1.png)

Set breakpoints on functions like **GetDlgItemText**. This is needed so the debugger stops at the right moment when checking text input.

![1.png]({{site.baseurl}}/images/crackme1/2.png)

Press the **Run** button so then the debugger steps at the right spot.

![1.png]({{site.baseurl}}/images/crackme1/6.png)

Press **Step over** to step through the code in the crackme's execution. The code will first check if the entered name is less than 5 and greater than 32 characters. If the entered name meets either of those conditions a error message will show.

![1.png]({{site.baseurl}}/images/crackme1/3.png)

The following screenshot shows the first part of the serial algorithm.

![1.png]({{site.baseurl}}/images/crackme1/4.png)

The code computes a buffer which is used for the final stage of the serial check. Basic arithmetic is used to compute the buffer as explained by the comments by the debugged code.

![1.png]({{site.baseurl}}/images/crackme1/5.png)

The final checks are computed by this code. Using the computed buffer from earlier as a guide, some basic arithmetic is used to check against the following code. If each portion of the serial entered matches what was computed by this code, the serial is considered valid.

## Keygen source code.

[Source code to the keygen is here.](https://github.com/mountnside/crackme_solutions/blob/master/keygenned/algo/d2k2_crackme01.c)

The code implemented in the keygen reimplements the x86 assembly code in C and also has modifications to the algorithm to generate portions as needed to fit the criteria used in the last section of the crackme.
