---
layout: post
published: true
title: Keygenning diablo2oo2's crackme 9
---
## Overview

The following tutorial will document how to program a keygenerator for diablo2oo2's
ninth crackme.

Functions, some addresses and variables are labeled to make the tutorial easier to follow.

The crackme can be found [here.](https://github.com/mountnside/crackme_solutions/blob/master/crackmes/d2k2_crackme9.zip)


## Reverse engineering the crackme.

Load up the executable in x32dbg. 
The debugger should load the executable at the following code:

![1.png]({{site.baseurl}}/images/crackme9/1.PNG)

Run the crackme in x32dbg.

![2.png]({{site.baseurl}}/images/crackme9/2.PNG)

There is two needed fields, a username and a serial field.

Place a breakpoint on the DLGPROC callback for the dialog box which is at 0x401040. This can be found by following the parameters being pushed onto the stack at the call DialogBoxParam().

(there exists some plugins for x32/x64dbg for easy identifying of parameters in Win32 calls)

In the callback 0x00401040, there is several things of note:

![3.png]({{site.baseurl}}/images/crackme9/3.PNG)

1. There is several GetDlgItemTextA calls for getting the text from the two fields.
2. There is a subroutine at 0x004010BE which uses the data from the two text boxes. This is our main call.

![4.png]({{site.baseurl}}/images/crackme9/4.PNG)


Place a breakpoint at 0x00401040, enter any name into the name box (I used "mountnside"), enter a serial
(I used the serial shown above) and run...

![5.png]({{site.baseurl}}/images/crackme9/5.PNG)

In this routine.....
* The username and serial are gathered and checked for length and sanity against a hardcoded table.

![6.png]({{site.baseurl}}/images/crackme9/6.PNG)

* A new table is made using the username as a basis. The length is checked, and from there, a new table that wraps around to be the same length is made. For example if there is the initial table of:
"0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ", the username "mountnside" would have a table of "stuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789abcdefghijklmnopqr" because of the position used to start from to make a contiguous table of 62 characters.

![7.png]({{site.baseurl}}/images/crackme9/7.PNG)
![8.png]({{site.baseurl}}/images/crackme9/8.PNG)

* A key is created based on the name and the newly created table. Some basic math is used to pick points in the table to make the new "key" from.

![9.png]({{site.baseurl}}/images/crackme9/9.PNG)


* The serial is verified using a multiple step process.

![10.png]({{site.baseurl}}/images/crackme9/10.PNG)

- A character from the generated key is checked for its position in the table made earlier.
- **Another** table is created using the character picked from the key as a starting point.
- The position in the newly made table from a character of the entered serial is found.
- A character in this newly made table is picked using the position in the previous step is picked.
- The newly picked character is now checked against the current username. If it doesn't match its rejected. Otherwise the checks keep going through until it succeeds.

This way of checking for things strongly correlates to [how the Vinigere cipher works.](https://en.wikipedia.org/wiki/Vigen%C3%A8re_cipher)

It uses a ring of interconnecting strings to encrypt and decrypt data, according to subsitution tables and passwords. 

A means to generate such a valid key for this target can be the following source code:

```
const BYTE validchar_tbl[] = "0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ";
#define validchar_tbllen 62

void process_serial(char* name, char* serial_out)
{
	int seriallen = strlen(name);
	char* ciphertext = (char*)malloc(seriallen + 1);
	memset(ciphertext, 0, seriallen + 1);
	int shlvar = (seriallen << 2) >= 0x3C ? 0x1E : seriallen << 2;
	unsigned char subtable1[validchar_tbllen] = { 0 };
	for (int i = 0; i < validchar_tbllen; i++)
		subtable1[i] = validchar_tbl[(shlvar++ % validchar_tbllen)];
	uint8_t AL = 0, DL = 0;
	DL = _rotl8(name[0], 3);
	for (int i = 0; ; i++)
	{
		unsigned char ciphertable[validchar_tbllen] = { 0 };
		int j = i + 1;
		AL = (name[i] ^ name[j]) + DL;
		DL += AL;
		AL = subtable1[AL % validchar_tbllen];
		int offset = 0;
		for (int k = 0; k < validchar_tbllen; k++)
			if (subtable1[k] == AL)
				offset = k;
		for (int k = 0; k < validchar_tbllen; k++)
			ciphertable[k] = subtable1[offset++ % validchar_tbllen];
		for (int k = 0; k < validchar_tbllen; k++)
			if (subtable1[k] == name[i])
				ciphertext[i] = ciphertable[k];
		if (j == seriallen)break;
	}
	wsprintf(serial_out, "%s", ciphertext);
	free(ciphertext);
}
```



## Keygen source code.

[Source code to the keygen is here.](https://github.com/mountnside/crackme_solutions/blob/master/keygenned/algo/d2k2_crackme09.c)

MSVC2019 is used to compile. It should compile out of the box. 

Feel free to use the template for your own crackme keygens.




