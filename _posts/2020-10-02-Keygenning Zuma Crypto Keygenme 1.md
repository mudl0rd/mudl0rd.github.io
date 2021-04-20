---
layout: post
published: false
title: Keygenning Zuma555's Crypto-Keygenme #1
---
## Overview

The following tutorial will document Zuma555's first cryptography keygenme, intended for beginner cryptographers.

This tutorial assumes you have some x86 assembly reversing knowledge and experience in x64dbg usage.

The crackme can be found [here.](https://github.com/mudlord/crackme_solutions/blob/master/crackmes/d2k2_crackme4.zip)

## Tools

You will need only x64dbg.

## Analysis

![1.png]({{site.baseurl}}/images/zumacryptokg/1.PNG)

Load up the crackme in x64dbg. 

![1.png]({{site.baseurl}}/images/zumacryptokg1/2.PNG)

The serial and username are loaded from user input into buffers.

![1.png]({{site.baseurl}}/images/zumacryptokg1/3.PNG)

A whole lot of BS logic is done, because at the end of the junkcode, ```EAX``` is overwritten anyway with the return value from lstrlenA.

![1.png]({{site.baseurl}}/images/zumacryptokg1/4.PNG)

![1.png]({{site.baseurl}}/images/zumacryptokg1/5.PNG)

Some basic mathematics is done on the value in EAX (which contains the username length)

![1.png]({{site.baseurl}}/images/zumacryptokg1/6.PNG)

The serial is split into two different buffers each seperated by a "-" character. 

![1.png]({{site.baseurl}}/images/zumacryptokg1/7.PNG)

Its contents are then transmuted into numeric values based on the characters in the string, using "XZMAJK" as the string.
For example, if the first character in the serial buffer is a "X", it is converted to a "0", if its a "Z" its converted to a "1" and so on.



There is some oddities to this particular code due to some bugs in the crackme.

In the first iteration of the loop shown, EDX starts at zero and is decremented by  ```dec dl```, so DX is now "0xFF" thus there is a out-of-bounds string read on the serial string, so DL starts at ```0xFF``` and wraps to zero. Due to this, the CL register also is equal to 0 and has no bearing on the resulting calculations. Also, due to another bug in the crackme, ESI is not equal to zero and so is set to the address of a subroutine in the crackme which is ```0x00401065```.

The original intent of the code was to loop over the characters in the entered string and do some very basic maths with the ASCII values. But due to the aforementioned bugs, only 7 characters are used in the calculation.

After the loop a XOR is done on the value in ESI with ```0x8EF21B```. The result of this calculation is used to alter the JMP address at ```0x00402D16``` from 
![1.png]({{site.baseurl}}/images/crackme4/5.PNG) 

to 

![1.png]({{site.baseurl}}/images/crackme4/6.PNG) 

with the entered serial of "12345678"

![1.png]({{site.baseurl}}/images/crackme4/7.PNG) 


The rest of the code is some more maths on the XORed value as well as a jump to a unknown memory location.
One interesting memory location is ```0x00401193``` which sets a serial box to the text buffer which is altered near the JMP's location.

After some experimenting at the JMP location, I found ```0xFFFFE479``` is the hex value of the machine code of the JMP instruction jump to where the serial verification message is shown . To get EDI to be that value though, the ASCII characters need to form a sum which is ```0xFFFFE479``` XORed by ```0x8EF21BD``` using the algorithm shown earlier.

To do this I wrote a small bruteforcer to run through ASCII strings until that value is met, using the method the crackme uses to calculate the rolling sum:

```
#define shitcalc(var) {\
    regconst += serial_buf[var]; \
    regconst *= regconst; \
} \
  

void process_serial(char *name, char *serial_out) {
  BYTE serial_buf[9] = {'0', '0', '0', '0', '0', '0', '0', '0', 0x0};
  //the good boy address XORed by this.
  signed int check_dword = 0xFFFFE479 ^ 0x08EF21BD;
  // pad buf with zero in case it fails.
  bool done = false;
  int num_ser = 0;
  FILE *pFile = fopen("keys.txt", "w");
  BYTE *ser_ptr = serial_buf;
  while (!done) {
    // the bruteforce
    (*ser_ptr)++;
    if (*ser_ptr == 'Z') {
      // limit range to usable ASCII keyspace
      *ser_ptr++ = '0';
      if (*ser_ptr == 0) {
        fclose(pFile);
        wsprintf((char *)serial_out, "Nothing found.");
        return;
      } else
        continue;
    } else {
      ser_ptr = serial_buf;
      //nonsense shit in ESI. 0x00401065 times itself
      signed int regconst = 0x338cc7d9;
      shitcalc(0);
      shitcalc(1);
      shitcalc(2);
      shitcalc(3);
      shitcalc(4);
      shitcalc(5);
      shitcalc(6);
      // right regconst?
      if (regconst == check_dword) {
        fprintf(pFile, "%s\n", serial_buf);
        num_ser++;
      }
      if (num_ser == 6)
        done = true;
    }
  }
  fclose(pFile);
  wsprintf((char *)serial_out, "Serials listed in \"keys.txt\"!");
}
```

Using the following serial "<LT;S610", which was found:


![1.png]({{site.baseurl}}/images/crackme4/9.PNG) 
![1.png]({{site.baseurl}}/images/crackme4/10.PNG)
![1.png]({{site.baseurl}}/images/crackme4/11.PNG)  
![1.png]({{site.baseurl}}/images/crackme4/12.PNG) 




