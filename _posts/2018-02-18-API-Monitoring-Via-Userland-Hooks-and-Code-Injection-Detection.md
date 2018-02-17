---
layout: post
title: API Monitoring via Userland Hooks and Code Injection Detection
---

# API Monitoring via Userland Hooks and Code Injection Detection

## About This Paper

The following document is a result of self-research of malicious software (malware) and its interaction with the Windows Application Programming Interface (WinAPI). It details the fundamental concepts behind how malware is able to implant malicious payloads into other processes and how it is possible to detect such functionality by monitoring communication with the Windows operating system. The notion of observing calls to the API will also be illustrated by the procedure of _hooking_ certain functions which will be used to achieve the code injection techniques.

**Disclaimer**: Since this was a relatively accelerated project due to some time constraints, I would like to kindly apologise in advance for any potential misinformation that may be presented and would like to ask that I be notified as soon as possible so that it may revised. On top of this, the accompanying code may be under-developed for practical purposes and have unforseen design flaws.

## Introduction

In the present day, malware are developed by cyber-criminals with the intent of compromising machines that may be leveraged to perform activities from which they can profit. For many of these activities, the malware must be able survive out in the wild, in the sense that they must operate covertly with all attempts to avert any attention from the victims of the infected and thwart detection by anti-virus software. Thus, the inception of stealth via code injection was the solution to this problem.

----

# Section I: Fundamental Concepts

## Inline Hooking

Inline hooking is the act of detouring the flow of code via _hotpatching_. Hotpatching is defined as the modification of code during the runtime of an executable image<sup>[1]</sup>. The purpose of inline hooking is to be able to capture the instance of when the program calls a function and then form there, observation and/or manipulation of the call can be accomplished. Here is a visual representation of how normal execution works:

```
Normal Execution of a Function Call

+---------+                                                                       +----------+
| Program | ----------------------- calls function -----------------------------> | Function |  | execution
+---------+                                                                       |    .     |  | of
                                                                                  |    .     |  | function
                                                                                  |    .     |  |
                                                                                  |          |  v
                                                                                  +----------+

```

versus execution of a hooked function:

```
Execution of a Hooked Function Call

+---------+                       +--------------+                    + ------->  +----------+
| Program | -- calls function --> | Intermediate | | execution        |           | Function |  | execution
+---------+                       |   Function   | | of             calls         |    .     |  | of
                                  |       .      | | intermediate   normal        |    .     |  | function
                                  |       .      | | function      function       |    .     |  |
                                  |       .      | v                  |           |          |  v
                                  +--------------+  ------------------+           +----------+

```


This can be separated into three steps. To demonstrate this process, the WinAPI function [MessageBox](https://msdn.microsoft.com/en-us/library/windows/desktop/ms645505(v=vs.85).aspx) will be used.

1. Hooking the function

To hook the function, we first require the intermediate function which **must** replicate the targetted function. Microsoft Developer Network (MSDN) defines `MessageBox` as the following:

```c
int WINAPI MessageBox(
  _In_opt_ HWND    hWnd,
  _In_opt_ LPCTSTR lpText,
  _In_opt_ LPCTSTR lpCaption,
  _In_     UINT    uType
);
```

So the intermediate function may be defined like so:

```c
int WINAPI HookedMessageBox(HWND hWnd, LPCTSTR lpText, LPCTSTR lpCaption, UINT uType) {
    // our code in here
}
```

Once this exists, execution flow has somewhere to where the code will be redirected or _jumped_. To actually _hook_ the `MessageBox` function, the first few bytes of the code can be _patched_ (keep in mind that the original bytes must be saved so that the function may be restored for when the intermediate function is finished). Here is the original assembly instructions of the function as represented in its corresponding module `user32.dll`:

```asm
; MessageBox
8B FF   mov edi, edi
55      push ebp
8B EC   mov ebp, esp
```

versus the hooked function:

```asm
; MessageBox
68 xx xx xx xx  push <HookedMessageBox> ; our hooked function
C3              ret
```

Here I have opted to use the `push-ret` combination instead of an absolute `jmp` due to my past experiences of it not being reliable for reasons I have yet to discover. `xx xx xx xx` represents the little-endian byte-order address of `HookedMessageBox`.

2. Capturing the function call

When the program calls `MessageBox`, it will execute the `push-ret` and effectively jump into the `HookedMessageBox` function and once there, it has complete control over the paramaters and the call itself. To replace the text that will be shown on the message box dialog, the following can be defined in `HookedMessageBox`:

```c
int WINAPI HookedMessageBox(HWND hWnd, LPCTSTR lpText, LPCTSTR lpCaption, UINT uType) {
    TCHAR szMyText[] = TEXT("This function has been hooked!");
}
```

`szMyText` can be used to replace the `LPCTSTR lpText` parameter of `MessageBox`. 

3. Resuming normal execution

To forward this parameter, execution needs to continue to the original `MessageBox` so that the operating system can display the dialog. Since calling `MessageBox` again will just result in an infinite recursion, the original bytes must be restored (as previously mentioned).

```c
int WINAPI HookedMessageBox(HWND hWnd, LPCTSTR lpText, LPCTSTR lpCaption, UINT uType) {
    TCHAR szMyText[] = TEXT("This function has been hooked!");
    
    // restore the original bytes of MessageBox
    // ...
    
    // continue to MessageBox with the replaced parameter and return the return value to the program
    return MessageBox(hWnd, szMyText, lpCaption, uType);
}
```

## API Monitoring

...

----

### References:

* [1] https://www.blackhat.com/presentations/bh-usa-06/BH-US-06-Sotirov.pdf