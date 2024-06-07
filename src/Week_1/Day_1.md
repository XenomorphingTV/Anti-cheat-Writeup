# 1.1 Day 1 - 6/6/2024
Alright so to start off I decided to define certain things. This way I wouldn't be confused I made a file `src/Definitions` that should have all the definitions. I will try to keep it updated, but I am quite a hands on learner so I may have to go back and edit them later (was my intention anyways). Because I prefer hands on learning, I decided to just hop in right away and learn as I go along. I will fill out more chapters and concepts as I go along. 

Alright, so to get started I installed and prepared Ghidra<sup>[[2]](#sources)</sup> on my system. I then made sure to have EAC anti-cheat installed on my computer. They where installed in my `C:\Program Files (x86)\EasyAntiCheat` and contained 2 files, `EasyAntiCheat.exe` and `EasyAntiCheat.sys`. I dragged and dropped the `.exe` into my Ghidra and began the code analysis. After a few minutes I had some pseudocode and began analyzing it. 

## 1.1.1 Entry

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

## 1.1.2 Main

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

All of the windows function that I refrence are from the Microsoft Learn (MDSN)<sup>[[4]](#sources)</sup>, or I link them as a source. So make sure to check that the function exists on the website before having to look somewhere else.

Alright, back to `StartServiceCtrlDispatcherW`. According to Microsoft, this function connects the main thread of a service process to the [Service Control Manager (SCM)](../Definitions.md#service-control-manager-scm). SCM is a Windows system process that starts, stops and interacts with service processes. The reason an application may want to do this is because it allows the application to run in the background as a service, start automatically and with elevated privileges, and can be easier managed through the Services console or PowerShell. These all make sense as to why EAC would make itself as a service.

After checking the status code of the return, the next step is checking some variable to see if it is null or has a null terminated character (specifically a wide character).
```cpp
if ((local_180 == (LPCWSTR)0x0) || (*local_180 == L'\0')) {
```
 Because of that I assume it is some sort of string. Looking a little higher shows me that in fact it is a string, as the variable `local_180` is infact a pointer to the `envp` which is passed to main. The `envp` parameter is a pointer to an array of null-terminated strings that represent the values set in the user's environment variables<sup>[[5]](#sources)</sup>. I will rename `local_180` to `pEnvironment`.

 Next we have what I believe to be a jump target, but I don't know yet, so I will skip it. We move onto this line:

 ```cpp
cVar1 = FUN_0041c200();
 ```
This is our first custom function, so we will move into this before me move on with the main.

## 1.1.3 CheckAdminPrivileges() 
Alright, after a quick look I saw this peculiar thing on the first instruction line
```cpp
local_8 = DAT_004bb014 ^ (uint)&stack0xfffffffc;
```

I was struggling to fin out what it is, but after looking where the `DAT_004bb014` was used, I noticed something interesting. It was often passed to a function in areas where strings where used, and where always called just before the return. This led me to believe it is most probably the value that stores the security cookie. This is confirmed here:
```cpp
FUN_0046cbcb(securityCookie ^ (uint)&stack0xfffffffc);
```
If we take a closer look at the function we can see this:
```cpp
void __fastcall FUN_0046cbcb(int param_1)

{
  undefined1 in_stack_00000004;
  
  if (param_1 == DAT_004bb014) {
    return;
  }
  FUN_0046cfdb(in_stack_00000004);
  return;
}
```
We can see that it compares the parameter to the address that we saw earlier. So we can conclude this is the security cookie function compare. We don't need to go any further here as we already know what it does. 

Next we have this section of code:
```cpp
TokenHandle = &hLocalProcess;
hLocalProcess = (HANDLE)0x0;
DesiredAccess = TOKEN_QUERY;
ProcessHandle = GetCurrentProcess();
bSuccess = OpenProcessToken(ProcessHandle,DesiredAccess,TokenHandle);
if (bSuccess == 0) {
  GetLastError();
  pcVar3 = "[EAC Service] [Error] Failed to get process token: %i.";
}
```
We can see that the TokenHandle is set to the to the pointer of `local_24` before it is set to null. Then the request level is set to 8. I looked around and after opening the `winnt.h` header file, I saw the access levels defined on MDSN. 8 here equates to `TOKEN_QUERY`. We then get the current handle to our process and try to `OpenProcessToken` with all of the variables. The status is then recorded and checked for errors if failed. 

Now we get to something that looks like it's grabbing more information about the application:
```cpp
else {
  bSuccess = GetTokenInformation(hLocalProcess,TokenElevation,local_28,4,&local_2c);
  if (bSuccess != 0) goto LAB_0041c2ab;
  GetLastError();
  pcVar3 = "[EAC Service] [Error] Failed to get token info: %i.";
}
```
And indeed after looking looking up the function we can easily name the rest of the variables and know what it is doing. We are trying to get the elevation level of the anti-cheat to ensure it is running at admin. With this information we can deduce its a function to check anti-cheat is running as admin.

I end here today as I have been working for a while now. Tomorrow I will continue working through the main function

## 1.1.S Sources

1. Ghidra official website - https://www.ghidra-sre.org/

2. __security_init_cookie() - https://learn.microsoft.com/en-us/cpp/c-runtime-library/reference/security-init-cookie?view=msvc-170

3. Why Is The PE Entry Point Not The Same As Main - https://www.patreon.com/posts/why-is-pe-entry-61343353

4. Microsoft Learn (MDSN)- https://learn.microsoft.com/en-us/windows/

5. Argument decription of main function - https://learn.microsoft.com/en-us/cpp/c-language/argument-description?view=msvc-170