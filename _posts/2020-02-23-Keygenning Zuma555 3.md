---
layout: post
published: true
title: Keygenning Zuma's crackme 3
---
## Overview

The following tutorial will document how to program a keygenerator for Zuma555's
third crackme.

This tutorial assumes you have some basic reversing knowledge, as well as familiarity with unpacking basic packers with the ESP trick.

## Tools

You will need

* x64dbg
* SND Reverser Tool
* ProtectionID
* Keygener's Assistant

## Analysis

First of all, it would be handy to get an idea of what we are dealing with here, so we use
ProtectionID to scan for anything wrapping the crackme.

Scanning with ProtectionID yields the following:

![protectionid.png]({{site.baseurl}}/images/zuma3/protectionid.png)

Okay, so it uses ASPack to pack the EXE. Unpacking is simple if you know how. If you know
the [ESP trick](https://reverseengineering.stackexchange.com/questions/11490/unpacking-and-the-esp-trick), it will work. So, then unpack the executable.

From here we need to verify what sort of encryption algorithms, if any, is used on the
executable. We use Keygener's Assistant for this:

![keygener.png]({{site.baseurl}}/images/zuma3/keygener.png)

We get one hit for MD5 and one hit for AES-MARS.

Loading up the unpacked crackme in x64dbg we get the following

![keygener.png]({{site.baseurl}}/images/zuma3/entrypoint.png)

Looking for any text input calls we see the following:

![keygener.png]({{site.baseurl}}/images/zuma3/1.png)

The following code:

1) Clears some data buffers

2) Gets the entered username. If it checks against a certain string, it fails.

3) Otherwise the MARS encryption cipher code is initialized with a key

4) The username's length is checked

5) A running bytesum of all the bytes in the entered username is done and outputed
as a decimal string.

6) The decimal string is encrypted using AES-MARS in 128bits using ECB.

![keygener.png]({{site.baseurl}}/images/zuma3/2.png)

7) The length of the encrypted string is checked and the encrypted bytes are output as a sequence of decimal numbers.
For example: "DD 99 F3 40 E7 FA 25 64 CE B8 FC B8 0E 63 EB CB" becomes "1089706416802106-1191397-87376610"

8) Some maths is done on some ASCII characters of some hardcoded strings. This gives the string of "55600" when formatted.

9) The last few portions of the string from step 7 is concatenated into a new string. This in part with the string from step 8 will form a serial.

10) This serial is string checked against the entered one. If it matches, its a valid serial being entered.


## Keygen source code.

Using the above deduced info, the following is worked out to generate valid serials.

```
#include <stdbool.h>
#include <windows.h>
#include <stdint.h>

extern void _stdcall mars_setkey(DWORD* input, DWORD input_len);
extern void _stdcall mars_encrypt(DWORD* input, DWORD* output);

void init() {}

void process_serial(char *name, char *serial_out) {
  unsigned namelen = lstrlen(name);
  uint8_t* mars_key = "FEABCBFFFF183461";
  mars_setkey(mars_key, strlen(mars_key));
  //pad out blocksize to match block coming out, sym crypt 101
  uint8_t namestr[0x10] = {0};
  uint8_t mars_out[0x20] = { 0 };
  uint8_t mars_tbl[0x50] = { 0 };

  uint8_t magic[0x6] = { 0 };
  unsigned namesum = 0;
  for (unsigned i = 0; i < namelen; i++)
      namesum += name[i];
  wsprintf(namestr, "%lX", namesum);

  mars_encrypt(namestr, mars_out);

  uint32_t* mars_ptr = mars_out;
  uint8_t* marsstrptr = mars_tbl;
  marsstrptr += 6;
  for (unsigned i = 0; i < 0x20; i += 4)
  {
      wsprintf(&marsstrptr[i*2], "%d", *mars_ptr);
      mars_ptr++;
  }

  unsigned marstbloutlen = strlen(marsstrptr);
  magic[0] = mars_tbl[5 + marstbloutlen];
  magic[1] = mars_tbl[4 + marstbloutlen];
  magic[2] = mars_tbl[3 + marstbloutlen];
  magic[3] = mars_tbl[1 + marstbloutlen];
  magic[4] = mars_tbl[marstbloutlen];
  wsprintf(serial_out, "%s-55600",magic);
}
```
