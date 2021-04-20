---
layout: post
published: true
title: Reversing MaRKuS TH-DJM's crackme 1
---
## Overview

The following tutorial will document how to patch MaRKuS TH-DJM's crackme.

This tutorial assumes you have some basic reversing knowledge.

The crackme can be found [here.](https://github.com/mudlord/crackme_solutions/blob/master/crackmes/markus_th-djm_crackme1.rar)

## Tools

You will need

* x64dbg
* ProtectionID

## Analysis

First of all, it would be handy to get an idea of what we are dealing with here, so we use
ProtectionID to scan for anything wrapping the crackme.

Scanning with ProtectionID yields the following:

![protectionid.png]({{site.baseurl}}/images/markuscrack/protectionid.PNG)

Okay, so it doesn't use any obvious packer. 

Running the crackme without any debugger yields the following:

![protectionid.png]({{site.baseurl}}/images/markuscrack/4.PNG)
![protectionid.png]({{site.baseurl}}/images/markuscrack/5.PNG)

Obviously we have 2 goals:

1) Remove the nag screen.

2) Change the text displayed.

Loading up the crackme in x64dbg shows up this:

![protectionid.png]({{site.baseurl}}/images/markuscrack/1.PNG)


Essentially, the whole code section is giving memory [execute/read/write access](https://docs.microsoft.com/en-us/windows/win32/memory/memory-protection-constants). 
This allows for the executable's code section to be manually overwritten at will with other contents loaded elsewhere.

Then the whole crackme file is read into a buffer. The buffer is also given execute/read/write access.

![protectionid.png]({{site.baseurl}}/images/markuscrack/2.PNG)

The file is then read and written to from the buffer, constantly decrypting and encrypting as execution happens.

![protectionid.png]({{site.baseurl}}/images/markuscrack/3.PNG)
![protectionid.png]({{site.baseurl}}/images/markuscrack/7.PNG)

 If at any point a breakpoint is set on any point in the code execution process, any form of stepping happens or any form of EXE modification happens, the buffer gets corrupted and the executable aborts execution.

 One point of solace is that placing breakpoints at any place **outside** the code section doesn't affect the executable memory. Checking the imports used by the executable is at least a clue as to whats being done:

 ![protectionid.png]({{site.baseurl}}/images/markuscrack/8.PNG)

 Clearly the call [MessageBoxA](https://docs.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-messageboxa) is a clue as to what the crackme uses to display the message box "nag" message. And sure enough, placing a breakpoint by following the imported address shows thats indeed the call it uses:

  ![protectionid.png]({{site.baseurl}}/images/markuscrack/9.PNG)

  And the call [SetDlgItemTextA](https://docs.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-messageboxa) is the right call used to set the "registered to:" text:

   ![protectionid.png]({{site.baseurl}}/images/markuscrack/10.PNG)

   Placing breakpoints on these functions in the executable's memory space won't work normally on execution since the executable's contents are already encrypted and not decrypted yet, so surely there is a different way to solve the crackme.
 

## API hooking

So to solve the crackme: two things need to happen.

1) The nag message needs to not exist.

2) The "Registered to:" text needs to be altered.

There is a way to do this without touching the main memory space of the running executable, through [Windows API hooks](https://www.pelock.com/articles/intercepting-dll-libraries-calls-api-hooking-in-practice) via [DLL proxying](https://kevinalmansa.github.io/application%20security/DLL-Proxying/).

With this method its possible to alter how certain API calls we want to target are handled by the host application, which in this case is the crackme. 

 ![protectionid.png]({{site.baseurl}}/images/markuscrack/11.PNG)

 Using "winmm.dll" as the victim DLL, any audio playback functions used by the crackme are wrapped and called by a proxy DLL. At the initialization of this process, 2 API hooks are placed. These are:

 ```
typedef BOOL(WINAPI* dlgitems)
(HWND   hDlg,
	int    nIDDlgItem,
	LPCSTR lpString);

dlgitems settext = NULL;
BOOL _stdcall  hooksetwindow(
	HWND   hDlg,
	int    nIDDlgItem,
	LPCSTR lpString)
{
	
	return settext(hDlg,nIDDlgItem, "Cracked by mudlord, take that Markus ;)");
}

typedef int(WINAPI* MessageBoxA2)
(HWND   hWnd,
	LPCSTR lpText,
	LPCSTR lpCaption,
	UINT   uType
	);
MessageBoxA2 timeout = NULL;

int _stdcall  message(HWND   hWnd,
	LPCSTR lpText,
	LPCSTR lpCaption,
	UINT   uType
)
{
	return 0;
}

void DLL_patch(HMODULE base) {
	uBase = uAddr(base);
	PIMAGE_DOS_HEADER pDsHeader = PIMAGE_DOS_HEADER(uBase);
	PIMAGE_NT_HEADERS pPeHeader = PIMAGE_NT_HEADERS(uAddr(uBase) + pDsHeader->e_lfanew);
	auto hExecutableInstance = (size_t)GetModuleHandle(NULL);
	IMAGE_NT_HEADERS* ntHeader = (IMAGE_NT_HEADERS*)(hExecutableInstance + ((IMAGE_DOS_HEADER*)hExecutableInstance)->e_lfanew);
	SIZE_T size = ntHeader->OptionalHeader.SizeOfImage;
	DWORD oldProtect;
	VirtualProtect((VOID*)hExecutableInstance, size, PAGE_EXECUTE_READWRITE, &oldProtect);

	dprintf("Let's patch!\n");
	wchar_t szExePath[MAX_PATH];
	GetModuleFileName(nullptr, szExePath, MAX_PATH);
	dwprintf(L"szExePath: %s\n", szExePath);
	if (0 != wcscmp(wcsrchr(szExePath, L'\\'), L"\\crackme.exe")) {
		return;
	}
	uBase = uAddr(base);

	MH_Initialize();
	MH_CreateHookApiEx(L"user32.dll", "SetDlgItemTextA", &hooksetwindow, reinterpret_cast<LPVOID*>(&settext), NULL);
	MH_CreateHookApiEx(L"user32.dll", "MessageBoxA", &message, reinterpret_cast<LPVOID*>(&timeout), NULL);
    MH_EnableHook(MH_ALL_HOOKS);
}
 ```

 Thus, the API calls return what we want and are altered to specify different text on calling them.

 As a result:

 ![protectionid.png]({{site.baseurl}}/images/markuscrack/patched.PNG)

## Source code

 Source code to the patch is [here.](https://github.com/mudlord/crackme_solutions/tree/master/markus_crack) 




