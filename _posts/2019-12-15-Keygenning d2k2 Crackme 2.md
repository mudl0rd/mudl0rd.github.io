---
layout: post
published: true
title: Keygenning diablo2oo2's crackme 2
---
## Overview

The following tutorial will document how to program a keygenerator for diablo2oo2's
second crackme.

Functions, some addresses and variables are labeled to make the tutorial easier to follow.

![1.png]({{site.baseurl}}/images/crackme2/1.PNG)

Load up the crackme in x64dbg.

![1.png]({{site.baseurl}}/images/crackme2/2.PNG)

Immediately there is something suspicious. Looking in the memory map section of x32dbg there is **3** executable code sections, with one labeled "yC". Upon scanning the executable in ProtectionID, it yields the following....

![1.png]({{site.baseurl}}/images/crackme2/3.PNG)

There are signs it is protected with a protection called "Yoda's Crypter". This particular DRM obfuscates code which is then deobfuscated on launch while also using anti-debug methods to try to detect a debugger like x32dbg and then force closing the crackme.

![1.png]({{site.baseurl}}/images/crackme2/4.PNG)

Indeed, once all possible application exceptions are ignored by the debugger, if its run in a debugger, it closes itself! (it runs fine outside a debugger, though)

One good way in x32dbg to defeat easy anti-debug tricks is the "Hide PEB" function. This patches the [Process Environment block](https://ntopcode.wordpress.com/2018/02/26/anatomy-of-the-process-environment-block-peb-windows-internals/), so some antidebug APIs like "IsDebuggerPresent" fail to be useful. Which is good for us.

To do this, after you load the executable into x32dbg, go to the "Debug" menu, go to "Advanced" and then "Hide debugger". This will not work for advanced protections such as VMProtect, Enigma and Denuvo, but works for basic protections such as Yoda's Crypter.

Once this is done, and all exceptions are ignored. Run the crackme.

![1.png]({{site.baseurl}}/images/crackme2/5.PNG)

Immediately we are greeted with this. To see the code in x32dbg for the crackme, once the DRM it has has unpacked, go to the "memory map" tab and click the "code" entry listed under the crackme. Typically the first executable code section is first in the list.

![1.png]({{site.baseurl}}/images/crackme2/6.PNG)

Clicking should drop you here:

![1.png]({{site.baseurl}}/images/crackme2/7.PNG)

Immediately we see such suspicious things as "GetDlgItemText" as well as the allocation of bytes into a table. Set a breakpoint on the beginning of the table like so:

![1.png]({{site.baseurl}}/images/crackme2/8.PNG)

Enter some details into the crackme (make sure the serial string is 16 characters in length), and then step through the crackme.

![1.png]({{site.baseurl}}/images/crackme2/9.PNG)

As you can see, my serial matches 16 characters in length (the "cmp eax,10" compares the length of the entered string to 0x10, in hex), and so doesnt make the code jump to not where we want to go.

![1.png]({{site.baseurl}}/images/crackme2/10.PNG)

Debugging further, our entered serial is checked to see if its less than one character in length and greater than 8 characters. If it meets that criteria, it goes to the next part of the serial check:

![1.png]({{site.baseurl}}/images/crackme2/11.PNG)

This bit is rather simple. As shown by the picture, the table generated earlier is modified by the entered username. Each second bit in the table is modified to be a uppercase letter, which is made uppercase by some basic ASCII math.

![1.png]({{site.baseurl}}/images/crackme2/12.PNG)

This bit is a bit more complicated, and can be described as follows.

1) A bit from the table messed with early is grabbed.

2) A variable is then increased by the value picked from the table each time. Once this is done the next step is done.

3) The name length is multiplied by 0xFF. The variable from step 2 is also multiplied by namelength*OxFF. This value is then XORed with a constant.


4) This variable from step 2 is then byteswapped and made as text into a new buffer.

6) A character from this new buffer is grabbed and if below a certain value, is changed and added back to the new buffer.

7) A bit from the new buffer modifies a every second bit of the old buffer made from step one.

8) Step 7 is then repeated until its complete for every second character in the original buffer.

9) A bit from the entered serial is checked with a bit from the original buffer. Also a bit from each second bit of the old buffer is checked with the serial buffer, only this time, the serial must be equal to the second bit of the buffer+5.

10) If it all matches, the serial is considered valid.

## Keygen source code.

[Source code to the keygen is here.](https://github.com/mudlord/crackme_solutions/blob/master/keygenned/algo/d2k2_crackme02.c)

The code implemented in the keygen reimplements the x86 assembly code in C and also has modifications to the algorithm to generate portions as needed to fit the criteria used in the last section of the crackme.

A practical implementation of how to make valid serials for this crackme is also here:

```
void process_serial(char *name, char *serial_out)
{
	char magic_buf[10]={0};
	char tabl[] = "SJKAZBVTECGIDFNG";
	int namelen = strlen(name);
	int magic_dword = 0;

	for (int i=0,j=0; i < namelen;i++,j+=2)
		tabl[j]=toupper(name[i]);

    for (int i=0;i!=0x10;i++)
		magic_dword += tabl[i];
	magic_dword = _byteswap_ulong((magic_dword * (namelen * 0xFF)) ^ 0xACEBDFAB);
	wsprintf((char*)magic_buf,"%1X", magic_dword);

	for (int i = 0; i < 8 ; i++)
	{
		byte val = magic_buf[i];
		if(val < 0x3A)
			val+= 0x11;
		magic_buf[i] = val;	
	}

    for (int i = 0, j = 0; i < 0x10; i += 2, j++)
    {
        byte val = (magic_buf[j]) + 5;
        tabl[i + 1] = val;
    }

	wsprintf(serial_out,"%s", &tabl[0]);
}
```
