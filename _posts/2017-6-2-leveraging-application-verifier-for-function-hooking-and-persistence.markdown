---
author: actae0n
layout: post
title: Leveraging Application Verifier for Function Hooking and Persistence
tags: [Windows, Persistence]
---

## Introduction to Application Verifier ##
Function hooking is a powerful tool for attackers. The ability to inspect and modify data structures as they're passed between functions within a program gives an attacker lots of options for session riding, credential theft, parameter modification, etc. We're going to take a look at an excellent tool that is provided to us by Microsoft called Application Verifier, which can be leveraged to perform hooking(and persistence) in a trivial, yet powerful manner by letting us run code in the context of an arbitrary unmanaged(native) application.

The [MSDN page](https://msdn.microsoft.com/en-us/library/ms220948(v=vs.90).aspx) describes AppVerifier as such: 

```
Application Verifier assists developers in quickly finding subtle programming errors that can be extremely difficult to identify with normal application testing. Using Application Verifier in Visual Studio makes it easier to create reliable applications by identifying errors caused by heap corruption, incorrect handle and critical section usage.
```
Application Verifier loads a particular **Verification Provider** into a process as it starts up. This provider, which comes in the form of a DLL, should contain the tests that are to be run in the context of the application being tested. These "tests" can be arbitrary. The provider is loaded at application startup, right after NTDLL.dll; Because of this, we won't have access to managed code segments in our attack code because we cannot load the CLR (Common Language Runtime, AKA .Net's execution environment) this early. For most things, this isn't an actual problem, but it's good to know. 


So, we can load code into any application as it starts using a mechanism provided by the operating system? ```LD_PRELOAD``` anyone? Let's see how we can leverage this.

Overall, the process of getting our code launched is as follows:
* We create a DLL which serves as the Verification Provider. This DLL is *supposed* to run routines that verify the application that it is loaded into. It will contain our function hooks and other code.  We will drop this into ```%windir%\System32\``` or ```%windir%\SysWoW64\``` (depending on the bitness of the target application).
* We create a registry key with a few subvalues which sets a few flags (to enable Application Verifier in the right mode) and detailing the name of the library which will serve as the verification provider (our malicious DLL).
* When the application starts (before other libraries are loaded, with the exception of ```NTDLL```) our Verification Provider will be loaded and execute its code. The load reason will not be ```THREAD_ATTACH``` or ```PROCESS_ATTACH``` as it typically is when a DLL is loaded. The value of fdwReason in this case is 4, which an existing symbol does not exist for. We will just define as ```VERIFIER_LOADED``` ourselves for readability and convenience. 


Note that this method requires administrator privileges (a UAC bypass should happen before installing the verification provider and its accompanying registry entries), as we're writing to HKLM and performing a privileged copy into ```System32``` or ```SysWow64```.

## Demonstration ##
For the demonstration of these capabilities, we're going to inspect and log POST requests made by Firefox by hooking the PR_WRITE function. This will give us access to the data as it's being sent, just before it becomes encrypted with TLS and sent over the network. 

Details on PR_WRITE and other functions in the Netscape Portable Runtime can be found [here](https://developer.mozilla.org/en-US/docs/Mozilla/Projects/NSPR/Reference/PR_Write){:target="_blank"}.

You can parse these requests with regular expressions to extract usernames and passwords from POST requests sent to login pages, for example. These siphoned passwords or sensitive requests can be loggedto a 3rd party server. I won't go over writing the regular expressions or reporting aspect, as this article is meant to focus on the deployment and implementation of the hook. When the function is called, our code will execute before the real function and we will be able to inspect or modify the data before forwarding it to the real function. For ```PR_WRITE```, the data being written to the specified file descriptor(```fd```) is stored in the ```buf``` parameter. 


## The Code ##

Sometimes you'll be working with mostly opaque code (no source available) and have to deal with reverse engineering to determine structure types and contents, function prototypes, etc. This can suck. In this case howerver, the code we're looking for (the Netscape Portable Runtime headers) is [available for inspection and copypasta](https://dxr.mozilla.org/mozilla-central/source/nsprpub/pr/include){:target => blank}. We'll be using their headers to define the structures we're accessing. 


Note that when compiling this DLL, its bitness must match the target application. You must also disable the CLR and set the entrypoint to ```DllMain```. Not doing so will cause the verifier to fail to load.  If you decide to implement any assembly routines to be run, you should also then disable optimizations. Visual Studio makes doing these things fairly easy via tweaking project settings.

{% gist 0a7906cd6bab09e7870ae6bafca9725f %}

With our DLL compiled, we just copy it into the correct directory like so:


![DLL Dropped]({{ site.url }}/images/hooky_in_sys32.PNG)


Then create the key ```firefox.exe``` under ```HKLM\SOFTWARE\Microsoft\Windows NT\Current Version\Image File Execution Options\```. Add the following values:

![Registry Values]({{ site.url }}/images/regvals.PNG)

Then launch Firefox from WinDbg. 

![Firefox Before Debug Resume]({{ site.url }}/images/before_debug_resume.PNG)


Press ```g``` to release the breakpoint and let Firefox continue execution, then navigate to a website. I'll go to Google.com; We can see in the window that our traffic that is passing through PR_WRITE is being logged successfully. 

![A Test Request]({{ site.url }}/images/navigating_to_google.PNG)

Now let's do a test to see if we really do have access to sensitive traffic. I'll try to log into Twitter (with fake credentials of course).

![Logging into Twitter]({{ site.url }}/images/twitter_req.PNG)

Now let's inspect our traffic log file and search for these values:

![Found It!]({{ site.url }}/images/located_data.PNG)

Awesome. The selection in question is also trivial to write a regular expression for, so you could write a version of this hook that excludes all of the "junk" traffic and only reports the username and password values with a regular expression filter.


## With Respect To Persistence ##

Instead of installing a function hook, you could simply use this code execution point as a springboard to launch some other component of your malware. You can still use the ```DLL_PROCESS_ATTACH``` event as normal, and it's subject to all the same restrictions as it typically is. 



## Sources: ##

* [Reiley Yang's MSDN Blog Post](https://blogs.msdn.microsoft.com/reiley/2012/08/17/a-debugging-approach-to-application-verifier/){:target="_blank"}
* [EP_X0FF's KernelMode.info thread](http://www.kernelmode.info/forum/viewtopic.php?f=15&t=3418){:target="_blank"}
* [Alex Ionescu's Presentation](https://www.youtube.com/watch?v=pHyWyH804xE){:target="_blank"}
