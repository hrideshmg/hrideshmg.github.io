---
title: Reverse Engineering A Windows Driver For Linux
description: Turning the unsupported into supported
slug: reversing-turbo
date: 2025-04-30
categories: dev
draft: true
---
I was recently watching PewDiePie's [video](https://youtu.be/pVI_smLgTY0?si=Yj5UFrUWDxJg7mFV) on his experience switching to Linux and something he said reminded me of one of the reasons why I enjoy using this operating system so much—the absolute freedom you have. If something doesn't work the way you want it to or a certain feature is unsupported you can dig into the code and fix it yourself!

As promised on my [blog-post]({{<ref "lfx-mentorship/#enabling-turbo-support-on-my-laptop">}}) about the  Linux Kernel mentorship program, this article will be about my endeavours in trying to enable [turbo]({{<ref "lfx-mentorship/#enabling-turbo-support-on-my-laptop">}}) support for my laptop to Linux.

## How it all began
It all started one day when I was sitting in front of my laptop completely zoned out. My eyes glanced upon the keyboard and I caught myself staring at a key which usually opens an app to change performance profiles of the laptop.

Now, much to the surprise of nobody, this key did not do anything on Linux since the NitroSense app (as is the case with most proprietary software) only works on Windows. I still wondered if it was possible to change thermal profiles on Linux. To my surprise, someone had already made a [kernel module](https://github.com/JafarAkhondali/acer-predator-turbo-and-rgb-keyboard-linux-module) for this purpose! Unfortunately (or fortunately, since we wouldn't have this article otherwise :)) it only supported the Acer Predator Series of laptops, whereas my laptop was from the Nitro Series.

Regardless, it proved as a good starting point and while going through the source code, I realized that this module was nothing more than a modified fork of a module from the platform profile subsystem on the kernel tree.

## Let the tinkering begin! 🛠️
### Discovering WMI
After a brief skim through the project, I understood that things like RGB LEDs, fan profiles and certain other hardware related functionalities are often controlled through something known as [WMI](https://en.wikipedia.org/wiki/Windows_Management_Instrumentation), during this phase I also chanced upon a youtube [miniseries](https://www.youtube.com/watch?v=97-WNhUmoig&list=PLv2kA4LxAI4Dq2ic_hU9bdvxIzoz5SzBr) created by the author of the project which gave me some great insight into how this project was built.

In short, the WMI [interface](https://docs.kernel.org/wmi/acpi-interface.html) allows software to communicate with the hardware by sending certain commands. These commands are handled by special WMI entries defined in the ACPI tables stored in the system firmware. That's a whole lot of words just to say — system send command, hardware do thing.

### Playing around with WMI
First off I started by randomly tweaking a few values in the source code (*sidenote: you probably shouldn't be doing this, especially at the kernel level!*) and briefly got my fans to spin to their maximum speeds but the thermal profiles did not seem to budge, my CPU was still throttled at a respectable 3.2 GHz.<br>Regardless, this confirmed my assumptions regarding WMI and thus I booted into Windows to monitor WMI activity and sure enough, whenever I changed thermal profiles using the app, a WMI event was registered on the [Windows Event Viewer](https://en.wikipedia.org/wiki/Event_Viewer).

The monitor told me that two WMI functions - `SetGamingFanBehavior` and `SetGamingMiscSetting` were called for changing fan speeds and applying overclocks respectively. Just knowing this alone wasn't enough though, I also needed to know what inputs are fed into these methods so that they actually do something. Sadly, the monitor provided no means to track inputs.<br>

### WMI Explorer
Initially, I tried to do some trial and error using a tool I discovered called [WMI Explorer](https://github.com/vinaypamnani/wmie2) to manually invoke these functions but it didn't seem to do anything. I later realized that the only way to figure out what the inputs are is to reverse engineer the program which calls them.

I also made a small documentation [patch](https://web.git.kernel.org/pub/scm/linux/kernel/git/next/linux-next.git/diff/Documentation/wmi/driver-development-guide.rst?id=98e45f0d7b99ceac029913ce3a161154a8c4c4a7) during this time to mention this tool in the WMI driver development [guide](https://docs.kernel.org/wmi/driver-development-guide.html).

## Reversing the NitroSense app
Thus it began, my first foray into reverse engineering a real app. The NitroSense app was written in C# and thus I used dotPeek to decompile it. 
### Fan Speeds Cracked
While searching for the input values for the overclock WMI call, I chanced upon a function that was responsible for setting the fan speeds - 
```C
public static bool set_all_fan_mode(CommonFunction.Fan_Mode_Type mode_index)
{
  ulong intput = 9;
  switch (mode_index)
  {
    case CommonFunction.Fan_Mode_Type.Auto:
      intput |= 4259840UL;
      break;
    case CommonFunction.Fan_Mode_Type.Max:
      intput |= 8519680UL;
      break;
    case CommonFunction.Fan_Mode_Type.Custom:
      intput |= 12779520UL;
      break;
  }
  return ((int) WMIFunction.SetAcerGamingFanGroupBehavior(intput).GetAwaiter().GetResult() & (int) byte.MaxValue) == 0;
}
```

As we can see, input values of the WMI function are readily available here! There are three magic values which correspond to the three different fan modes. Was a bit surprised to the see the typo in the 'intput' variable though, kind of refreshing to know that even billion dollar companies have such mistakes in their code lol.

### In Search Of Overclocks..
The overclock function was sadly not as simple of an egg to crack. I traced through the GUI code for the app and narrowed it down to this particular function -
```C
public static async Task<int> set_operation_mode(int Operation_Mode)
{
  int output = -1;
  try
  {
    NamedPipeClientStream cline_stream = new NamedPipeClientStream(".", "PredatorSense_service_namedpipe", PipeDirection.InOut);
    cline_stream.Connect();
    output = await Task.Run<int>((Func<int>) (() =>
    {
      IPCMethods.SendCommandByNamedPipe(cline_stream, 30, (object) (uint) Operation_Mode);
      cline_stream.WaitForPipeDrain();
      byte[] buffer = new byte[9];
      cline_stream.Read(buffer, 0, buffer.Length);
      return BitConverter.ToInt32(buffer, 5);
    })).ConfigureAwait(false);
    cline_stream.Close();
    return output;
  }
  catch (Exception ex)
  {
    return output;
  }
}
```
Don't worry if it all looks like gibberish to you, it did to me as well when I first came across it. Since i was wholly unfamiliar with the Windows API, I took to ChatGPT to explain the code to me like a 5 year old. GPT explained that this function was creating something called a ["Named Pipe"](https://en.wikipedia.org/wiki/Named_pipe) which is used for inter-process communication.

Uh oh, this meant that the actual WMI call was was being made by a process on the receiving end of this pipe. Interestingly, I noted that the name of the pipe was 'PredatorSense_**service**', suggesting that the recipient process was a service.

While we are here, let me also comment that the `Operation_Mode` argument of `set_operation_mode()` took one of three values: 0, 1 or 4. Which makes sense considering that my lapop has three performance modes.

### The Final Gatekeeper
Sure enough, there was indeed a service in services.msc named `Predator Service`, the service started a process called `Pssvc.exe`. This particular program was written in C++, I initially used IDA to dissassemble it before remembering that I had pretty much 0 knowledge of assembly ;-;

That's when I remembered Ghidra, a reversing tool developed by the NSA, that I had used during my first CTF contest - inCTFj. Ghidra tries its best to produce a readable C-like program from the disassembled code.

After some heavy decompiling on the service file using Ghidra I finally found out that the value is read from the named pipe and then used to call a function from a function pointer table as follows:
```C
(*(code *)(&PTR_LAB_140052c90)[uVar14])(puVar5,puVar8,&local_250,&local_244);
```
Here, `&PTR_LAB_140052c90` refers to the following function pointer table:

<img src="function_table.png" width="480">

If you remember from [earlier](#in-search-of-overclocks), a command index of 30 was passed into the named pipe along with the Operation Mode and this corresponds to the function pointer that I've highlighted in the image. Here is the relevant section of the code which calls this particular function:

<img src="ghidra_decomp.png" width="480">

Yes it is one hell of a monstrosity, but after staring at it for a long time, I figured out that all it does is pass an array of bytes taken from the named pipe and their corresponding byte offsets to the function. You can observe this in the last highlighted line of code, `puVar5` is the array of bytes and `puVar8` are the byte offsets (I think anyways)
