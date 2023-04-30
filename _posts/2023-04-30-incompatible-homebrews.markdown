---
layout: post
title:  "Why some homebrew apps are incompatible with Aroma and how to fix them"
date:   2023-04-30 17:30:00 +0100
categories: homebrew
---

Aroma has been released for 8 months, and almost every day I get reports of people asking why their homebrew does not appear in the Wii U Menu or why their apps crash when exiting. In this blog post, I want to give you more details about why these things happen and how developers can fix their apps.

## Early days of launching homebrew
Back in 2016, when the homebrew launcher (HBL) for the Wii U launched, it was only possible to load homebrew specially designed for it. Any already-existing homebrew applications had to be ported. Today, these "HBL compatible homebrews" are commonly known as `.elf`-homebrew. They are statically linked to a specific address in memory (which was dedicated to the homebrew launcher), which means in order to execute them, they need to be copied to a specific position in memory. This meant it was impossible to run multiple `.elf` homebrews at the same time. Loading a second one, would simply overwrite the first one. (For example, it was impossible to use SDCafiine and TCPGecko at the same time without combining them into a single homebrew app.)

To execute these applications, the homebrew launcher hooked into the Mii Maker (the only system app with SD card access) and redirected code execution to the loaded `.elf` in memory. To provide further information to the homebrew apps (like the system version, or function pointers to important system functions), the HBL placed them in a dedicated area in memory. Almost all of the `.elf`-homebrews heavily rely on this information and won't run without it.

Over time, toolchains have improved (e.g. [wut](https://github.com/devkitPro/wut)) and are now able to create homebrew in the native binary format of the Wii U, also known as `.rpx`. Unlike `.elf` homebrew, which had to be loaded manually into memory, loading `.rpx` files is handled by the OS. The Homebrew Launcher hooked into the loader and replaced the data on the fly. Aroma redirects the file loading to the SD card on the IOSU side. But both don't have to care how the files are actually loaded or where the executable is stored. Additionally, `.rpx`-homebrews are dynamically linked, which means it's possible to load them anywhere in memory, making them much more future-proof.

## Not every .rpx is implemented properly
Each application on the Wii U can have two states. It can be in the foreground or the background. Whenever the Home Menu or any applet (like the Friend List, Browser, etc.) is running, the application will be in background state. To properly support this, applications can use the ProcUI library to manage their state. On every frame, the application checks whether it should leave the foreground, come back from the background, or exit. Only if this is implemented can the application react to things like pressing the home button on the gamepad or the power button on the console.

In the early days of `.rpx`-homebrew, not many applications actually implemented the ProcUI-loop. People barely understood it, and there was zero documentation about it. The only way to learn about this was by looking at existing code or by reverse engineering official applications. Even today, this knowledge is not really widespread.

Because of the way the Homebrew Launcher is injected into the Mii Maker and how it's loading `.rpx`-homebrew, it's actually fine to run homebrew without a ProcUI-loop with the Homebrew Launcher. The home button was used to exit to the Homebrew Launcher instead of opening the Home Menu. Developers had no real incentive to implement it, because everything (except shutting down the console by pressing the power button) was working and only a few apps did it properly. It only became a problem when these `.rpx` homebrews were loaded as installed titles: Often opening the Home Menu didn't work, and exiting softlocked the console.

# Compatibility with Aroma

How does all of this affect Aroma? As we have learned before: `.elf`-homebrews are hardcoded to be launched from a specific address in memory and rely on some magic values in memory. Unfortunately, that specific address is exactly the same address that Aroma uses for its module system. This means we have to decide between supporting the old `.elf`-homebrew from <= 2016 or having all the cool new features Aroma introduced. For me, the choice was obvious. Almost every `.elf` nowadays has been replaced by a `.rpx`/`.wuhb` homebrew or is now a plugin.

Technically, it might be possible to bring back support for `.elf`-homebrew. But it either means I have to rewrite parts of Aroma (to avoid the memory region used by `.elf`-homebrew) or every existing `.elf`-homebrew has to be adjusted (or at least re-compiled). At this point, it just makes much much much more sense to just port it to `.rpx` (or to be a plugin).


**TLDR: Aroma will probably never support `.elf` homebrew because launching them would mean overwriting parts of Aroma.**

With Aroma, the way of loading `.rpx` files also changed. Loading homebrew is now directly integrated into the Wii U Menu and every homebrew acts and appears like a real installed title on the system. So loading a `.rpx` with Aroma that doesn't implement the ProcUI-loop has the same issues as installing it as a channel: Opening the Home Menu doesn't work properly, and exiting would softlock the console.

Additionally, some older `.rpx`-homebrews are bundling exploits that overwrite parts of Aroma in memory. This results in undefined behavior or even crashes.

**TLDR: Some existing `.rpx` have not been implemented properly; they were just good enough to work with the Homebrew Launcher and now need to be updated.**

(It's the same for `.wuhb`-homebrew. Every `.wuhb` is basically like a `.zip` containing a `.rpx` and metadata like icons. Having a `.wuhb` doesn't mean it'll work in Aroma, it's all about the `.rpx`'s implementation inside the `.wuhb`.)

## How to make a `.rpx` compatible with Aroma
As a rule of thumb: every `.rpx` that would run as an installed title (and does not bundle any exploits) will work with Aroma.

This means it has to implement the ProcUI-loop properly. ProcUI is a set of functions to manage transitions between the different states of an application. Each application can either be in the **foreground** (when you actually see the application running) or in the **background** (when something else is in the foreground, like the Home Menu or an applet like the Browser). To react to requests from the OS, the application can use the ProcUI library. **If the application doesn't react to the request to transit to the background or exit, the OS is still waiting for a response, even if the application has actually already exited, resulting in a softlock.**

#### Check for ProcUI status
In order to have a responsive application and to react quickly to events, it should check for its current state **on every frame**.
The current state can be checked by calling `ProcUIProcessMessages()`:
- `PROCUI_STATUS_IN_FOREGROUND`: The application is in the foreground (default state). All system resources and hardware are available, and the application runs without any serious restrictions.
- `PROCUI_STATUS_IN_BACKGROUND`: The application is in the background and something else runs in the foreground (Home Menu overlay, Internet Browser, etc.). Background applications are heavily restricted: they lose access to MEM1 and the foreground bucket, get a small amount of CPU time on core 2 (all other threads are suspended), access to filesystems, and some network IO. They have no access to graphics, inputs, or user interaction of any kind.
- `PROCUI_STATUS_RELEASE_FOREGROUND`: The application is in the foreground, but the OS wants to put it in the background. This either happens when the user opens an applet like the Home Menu (via the home button), Friend List, Browser, etc. or the application closes (e.g. by pressing the power button of the console). The application needs to release all foreground-only resources like MEM1 memory or the Foreground Bucket. If it's ready to release the foreground, it needs to call `ProcUIDrawDoneRelease()`.
- `PROCUI_STATUS_EXITING`: The application is in the background, and the OS wants to exit it completely. When receiving this message, the application needs to clean up all resources it's using and can then exit safely by returning from the main function. ProcUI needs to be shutdown via `ProcUIShutdown()`.

#### How to deal with losing access to memory regions while in the background
While the application is running in the background, it only has limited access to system resources. This includes access to the MEM1-region and the foreground bucket. Before the application goes into the background, you need to release all the memory in those regions. When the application goes into the foreground, the memory can be allocated again.

ProcUI offers two solutions for this:
- A callback can be registered, which will be called once the application releases or acquires the foreground. An example implementation that frees/allocates MEM1 via callbacks can be found [here](https://github.com/devkitPro/wut/blob/4a98cd4797d3b87a9f38a3999e471d3eebd850f5/libraries/libwhb/src/gfx.c#L354).
- As an alternative solution, `ProcUISetMEM1Storage` and `ProcUISetBucketStorage` can be used. These functions take a buffer (which needs to be allocated from MEM2) which will be used to preserve MEM1/Bucket memory while the application is in the background.

#### How to handle switching to different applications
On the Wii U, you never just return from the `main` function when you want to exit the application. The OS always needs to know what to launch next, and exiting always happens via the ProcUI-loop. For most cases, you can use the [sysapp](https://wut.devkitpro.org/group__sysapp.html) library for this.

Example: If you want to open the Wii U Menu, simply call `SYSLaunchMenu()` **once**. This will trigger a ProcUI event, which the application will then react to.

#### Interesting links related to ProcUI:
- To switch to another application or to open an applet, you can use the [sysapp](https://wut.devkitpro.org/group__sysapp.html) library.
- An example of a ProcUI-loop implementation can be found in [launchiine](https://github.com/wiiu-env/launchiine/blob/bd31cbe4f4487851e6a2aa79ad30fba5b73107d3/src/Application.cpp#L133).
- To simplify things, [wut](https://github.com/devkitPro/wut) provides some wrapper functions (`WHBProcIsRunning`) to handle ProcUI for you. Example implementations can be found in the [wut samples](https://github.com/devkitPro/wut/tree/master/samples).
- See the [wut documentation](https://wut.devkitpro.org/group__proc__ui__procui.html) for further information about the ProcUI library.

## Conclusion

In conclusion, `.elf`-homebrews are not compatible with Aroma for technical reasons, and the ProcUI library plays a crucial role in ensuring compatibility of `.rpx` homebrew apps. Without implementing the ProcUI-loop, homebrew apps won't be able to properly manage their foreground and background states. This results in a lack of responsiveness to user actions like pressing the home button or power button, which results in softlocking on exit. While it may have been fine to run these apps with the Homebrew Launcher, these applications are now causing trouble when launched in Aroma. It's easier to fix these few applications instead of putting many hours into supporting old legacy homebrew, which can be easily used by dualbooting into [Tiramisu](https://tiramisu.foryour.cafe/).

If you're a developer and have questions about fixing apps to be compatible with Aroma, feel free to ask on the [Aroma Discord](https://discord.com/invite/bZ2rep2).
