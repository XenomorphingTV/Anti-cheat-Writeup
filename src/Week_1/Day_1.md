# Day 1 - 6/6/2024
Alright so to start off I decided to define certain things. This way I wouldn't be confused I made a file `src/Definitions` that should have all the definitions. I will try to keep it updated, but I am quite a hands on learner so I may have to go back and edit them later (was my intention anyways). Because I prefer hands on learning, I decided to just hop in right away and learn as I go along. I will fill out more chapters and concepts as I go along. 

Alright, so to get started I installed and prepared Ghidra<sup>[[2]](#sources)</sup> on my system. I then made sure to have EAC anti-cheat installed on my computer. They where installed in my `C:\Program Files (x86)\EasyAntiCheat` and contained 2 files, `EasyAntiCheat.exe` and `EasyAntiCheat.sys`. I dragged and dropped the `.exe` into my Ghidra and began the code analysis. After a few minutes I had some pseudocode and began analyzing it. 

## Entry

Below is the entry function. It calls 2 functions before returning out.
```cpp
void entry(void) {
  ___security_init_cookie();
  __scrt_common_main_seh();
  return;
}
```
The first function is `__security_init_cookie()`. This function initializes a [global security cookie]() which is used for buffer overrun-protection.<sup>[[2]](#sources)</sup>

For the second function, I found no official documentation. I had to go looking in other areas to find this information. I found a nice thread that explained why this<sup>[[3]](#sources)</sup>, and this is because MSVC compiler does this. Therefore we can conclude that EAC is written in MSVC.

After enterint the `__scrt_common_main_seh()` I looked around for a little bit, and then I quickly found this bit here:
```cpp
unaff_ESI = FUN_0041ea40(0x400000,0,pWVar7);
```
It seems to match the 3 passed arguments that MSVC compiler expects from the main. I decided to renaim this function to `Main` after confirming.

## Main

After entering the function, I scrolled around for a little whiel trying to find anything interesting. I found some text that seemed to confirm that this function was indeed the main, because it was dealing with command line arguments
```cpp
FUN_0041e640(local_2c,"[EAC Setup] [Error] CommandLineToArgvW failed with %i.");
```

I assumed this was some sort of logging function, but I would take things sequentially. After scrolling back up, the first thing I saw was a bunch of variables being initialized. I had no clue of knowing what they where at the moment, so i decided to go look for some functions I could start looking at.

Looking at one of the first instructions, I see this:
```cpp
BVar2 = StartServiceCtrlDispatcherW(&local_198);
```
The BVar2 here is a status code that is checked for success right under it. I decided to rename the variables since this was a documented function. After doing so, I went to take a closer look at what the function does. 

All of the windows function that I refrence are from the Microsoft Learn<sup>[[4]](#sources)</sup>, or I link them as a source. So make sure to check that the function exists on the website before having to look somewhere else.

Alright, back to `StartServiceCtrlDispatcherW`. According to Microsoft, this function connects the main thread of a service process to the [Service Control Manager (SCM)](../Definitions.md#service-control-manager-scm).

## Sources

1. Ghidra official website - https://www.ghidra-sre.org/

1. __security_init_cookie() - https://learn.microsoft.com/en-us/cpp/c-runtime-library/reference/security-init-cookie?view=msvc-170

2. Why Is The PE Entry Point Not The Same As Main - https://www.patreon.com/posts/why-is-pe-entry-61343353

3. Microsoft Learn - https://learn.microsoft.com/en-us/windows/