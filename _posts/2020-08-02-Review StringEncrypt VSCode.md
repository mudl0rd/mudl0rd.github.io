---
layout: post
published: true
title: Review - "StringEncrypt" for Visual Studio Code
---
## Overview

### Note: a review code was supplied by the author of StringEncrypt

Sometimes its needed for one reason, or another, to obfuscate strings in your code.

There is various string classes around which can "encrypt" strings such as [skCrypter](https://github.com/skadro-official/skCrypter) and [xorstr](https://github.com/JustasMasiulis/xorstr).

However, these rely on modern C++ features. They do not support encryption of strings in different programming languages or even limited C++ usage. 

[StringEncrypt](https://www.stringencrypt.com/) aims to alleviate this with support for:

- C/C++
- C#/VB.NET
- MASM/FASM assembly
- Delphi/Pascal
- Java/Javascript
- Python
- Ruby 
- AutoIt
- PowerShell
- Haskell

Using StringEncrypt in Visual Studio Code is a breeze. Simply click the string to be obfuscated and right-click it.

![stringenc.png]({{site.baseurl}}/images/stringenc/1.PNG)

Then you specify the label to use for the encrypted string.

![stringenc.png]({{site.baseurl}}/images/stringenc/2.png)


![stringenc.png]({{site.baseurl}}/images/stringenc/3.PNG)

The string is then sent to a server to be encrypted via HTTPS, so the system works via SaaS, for the current source code filetype. The VSCode plugin then takes the response from the server and gets a encoded string, in the form of code to decrypt the string. Each string is unique and different for the same input.

![stringenc.png]({{site.baseurl}}/images/stringenc/4.PNG)

So far for me the plugin works well enough for these purposes. Each license of the code nets a certain amount of uses which can be set in the VSCode plugin options. The website itself can be used to encode various strings complete with decryption routines given.

If you ever need to encrypt strings in your application and have a limit to what programming language you can use, you cannot go past this encryption service.


