---
layout: post
published: true
title: Keygenning Yonkie's crackme 2
---
## Overview

The following tutorial will document how to program a keygenerator for Yonkies's
second crackme.

Functions, some addresses and variables are labeled to make the tutorial easier to follow.

The crackme can be found [here.](https://github.com/mudlord/crackme_solutions/blob/master/crackmes/yonkie_crackme.zip)

Scanning with YARA-GUI yields the following:

![1.png]({{site.baseurl}}/images/yonkie2/1.PNG)

So there is obvious signs of AES/Rijindael encryption usage as well as a cyclic redundancy check.

In the crackme package, there is 2 keyfiles, both of which are valid keyfiles. Our goal though
is to make our own keyfile which is loaded by the crackme and treated as valid.

Load up the crackme in x64dbg, and set the commandline to load "key1.dat". This can be done by clicking "File" and then clicking "Change Command Line". You will need to unload and then reload the crackme in x32dbg to get this to work.

![1.png]({{site.baseurl}}/images/yonkie2/2.PNG)

Looks like a standard MSVC entrypoint, so looking at the strings in the executable yields:

![1.png]({{site.baseurl}}/images/yonkie2/3.PNG)

This yields much more meaningful finds. Immediately the strings such as "Licensed name" and "key error" are suspicious, so place breakpoints on them all, as well as the string "Can't open this file". Placing some breakpoints on some apis such as fopen() wouldn't go astray either.

![1.png]({{site.baseurl}}/images/yonkie2/4.PNG)

The function at 0x0040162 seems to contain the logic for the crackme. Previously, the name and serial are grabbed from the keyfile using [regular expressions](https://en.wikipedia.org/wiki/Regular_expression).

![1.png]({{site.baseurl}}/images/yonkie2/5.PNG)

Tracing through this code, the serial string is checked and the values from the serial string are converted to hexadecimal, from ASCII to be in a buffer.

![1.png]({{site.baseurl}}/images/yonkie2/6.PNG)

The data in the buffer is then checked so that the first 12 bytes in the buffer add up to certain check values. In C this would equate as the following:

```
 uint32_t hash = 0;
    for (uint8_t i = 0; i < 12; i++)
      hash += buffer[i];
    if ((buffer[0x0E] == (hash >> 8)) && (buffer[0x0F] == LOBYTE(hash)))
	{
		// continue algorithm
	}
	else
	{
		//fail and exit here
	}

```

If the checks succeed the code goes further...

![1.png]({{site.baseurl}}/images/yonkie2/7.PNG)

A buffer made up of 16 bytes made by the C function "rand()" is created. It seeds the random number generator using the function "srand" with the seed 0x12345678. Upon some further reversing by me, it uses this buffer as a decryption key for the serial. The serial number is then decrypted using [128-bit AES](https://en.wikipedia.org/wiki/Advanced_Encryption_Standard). 

![1.png]({{site.baseurl}}/images/yonkie2/8.PNG)

The username is then checked using a [32-bit cyclic redundancy check](https://en.wikipedia.org/wiki/Cyclic_redundancy_check), which operates per-bit. In C the CRC algorithm's code is like this:

```
uint32_t crc32(int32_t namelen, uint8_t *buffer) {

  if (namelen <= 0) {
    return 0;
  }

  uint32_t result = 0;
  for (int i = 0; i < namelen; i++) {
    int bitcount = 8;
    result ^= (buffer[i] ^ 0xffffffff);

    do {
      uint32_t calc = -(result & 1) & 0xedb88320;
      result = result >> 1 ^ calc;
      bitcount--;
    } while (bitcount != 0);

    result ^= 0xffffffff;
  }
  return result;
}
```

The value gleaned from this algorithm, is checked against a portion of the decrypted buffer. If it matches the algorithm continues to check the "magic" of the decrypted buffer, which is yet another portion of it. If it matches against what is in the buffer, it continues to show things such as feature flags and the "serial" of the decrypted buffer.

Through some analysis the format of the keybuffer is of the following

```
struct keydata_format {
  uint32_t checksum;
  uint16_t feature_flags;
  uint16_t magic;
  uint32_t serial;
  uint32_t random;
};
```

The format of the keyfile used is of the following:



N=(username) K=(encrypted buffer matching format being checked)

For example, one of the example keyfiles has the following:



N=Yonkie K=6A5327F384088CC003115600DC430419

## Keygen source code.
[Source code to the keygen is here.](https://github.com/mudlord/crackme_solutions/blob/master/keygenned/algo/yonkie_crackme2.c)

Using the above deduced info, to generate a valid serial is done by the following method

1) Create a 16 byte buffer.


2) CRC32 the username using the exact same algorithm as the crackme and place the data in the first 4 bytes of the buffer


2) Set the feature flags to 0xFFFF, and place in the next 2 bytes in the buffer.


3) Place the magic constant 0x1979 into the next 2 bytes of the buffer.


4) Create a random serial number and place in next 4 bytes of the buffer.


5) Set the next 4 bytes as 0.


6) Encrypt the buffer using the AES key we found out.


7) Do a running addition of the first 12 bytes of the encrypted buffer.


8)


```
if ((buffer[0x0E] == (hash >> 8)) && (buffer[0x0F] == LOBYTE(hash)))
```


If the last 2 bytes do not match the following criteria, repeat steps 2 to 7 until the criteria is passed, only this time incrementing the last 4 bytes by 1 each time until the buffer check passes.

9) Format the bytes into a 32 character ASCII string.


10) Write to file the string:

```
sprintf(serial_out, "N=%s K=%s", name, buffer_string);
```