---
layout: post
title:  "Creating a homebrew environment on the Wii U - Part 2"
date:   2019-11-27 19:33:00 +0100
categories: homebrew
---
The [last blog entry]({{ site.baseurl }}{% link _posts/2019-11-20-new-environment-part1.markdown %})
  brought an overview of the internal structure of the Wii U. 
So that a homebrew environment can be designed, some requirements will be specified in this entry.
Afterwards the existing solutions will be considered and compared with these requirements.

## Requirements

1.  **The execution of own/unsigned software is possible:**  
    The homebrew environment should make it possible to execute any
    software written by the user. It should be possible to load and
    execute own software from an SD card.

2.  **The existing operating system is still used:**  
    The installed homebrew environment should use the existing operating
    system as a basis instead of starting its own operating system. It
    should still be possible to run official software. The applications
    or the operating system should be able to be modified by the
    homebrew environment.

3.  **The existing operating system can be manipulated:**  
    It should be possible to manipulate and extend the existing
    operating system if needed. For example, restricted access to the SD
    card can be removed and functions of the system libraries can be
    replaced.

4.  **Code execution in the background is possible**  
    During runtime, for example during the execution of a game, it
    should be possible to execute your own code in the background. This,
    together with requirement 3, should allow the software to be
    enhanced.

5.  **The homebrew environment is available after console start**  
    After a one-time setup, the homebrew environment should (optionally)
    immediately be usable. It should be possible to start in an homebrew
    environment in which the operating system modifications are
    available and own code execution is possible.

6.  **The homebrew environment can be implemented without hardware modifications**  
    Only software solutions should be necessary to achieve this homebrew
    environment. Physical modification of the hardware should not be
    necessary.

7.  **The homebrew environment can be used on the latest version of the operating system**  
    The homebrew environment should be executable on any *Wii U* that
    has not yet been modified. No specific versions of the operating
    system should be required, which cannot be reached via updates.

8.  **The homebrew environment should be easy to update and maintain**  
    Simple usage and updating of the homebrew environment should be
    achieved. Updates should be possible via a central point, such as
    replacing files on the SD card.

## Existing solutions

At first, we will look at existing solutions that are publicly available
for the Wii U. Subsequently, it shall be considered where the weaknesses
of these solutions lie and which of these solutions can be adapted to
meet the requirements set.

### Execution of own applications

Users can run homebrew applications on *Wii U* via the so-called
*Homebrew Launcher*. To use this, an initial entry point
into the system via an exploit is required. In addition, a kernel
exploit is required, which is integrated with the *Homebrew Launcher*.
While the *Homebrew Launcher* is executed via a browser exploit,
theoretically other entry points are also possible. The corresponding
homebrew applications are loaded from the SD card and then executed.

The *Homebrew Launcher* is divided into three components:

-   Installer
-   sd-loader
-   GUI

**Installer**: The installer is the part that is executed via a browser
exploit, for example. The installer performs various initial tasks to
prepare the system for loading additional code. The first task is to
obtain kernel privileges using a kernel exploit. This allows you to set
up your own memory area. This memory area has read, write, and execute
permissions. The Cafe OS does not use this memory area, so it can be
used as memory for the *Homebrew Launcher*. After setting up the memory
area, the sd-loader is copied to this area and remains there until the
console is turned off. The system is then manipulated so that the call
to an application first executes the sd-loader and then the actual
application.

**sd-loader**: Before each application is started, the sd-loader is
executed. Its task is to execute either the graphical interface of the
*Homebrew Launcher*, the actual application, or a homebrew application.
In the sd-loader, it is implemented that the graphical user interface is
called as soon as the user starts the *Mii Maker* application. It is
necessary to use the *Mii Maker* application for this as it is the only
pre-installed application that has access to the SD card. This access is
defined per application and is enforced via the *IOSU*.

**GUI**: The graphical user interface allows the user to select which
homebrew application is to be executed. If the user decides to execute
an application, it is loaded into the user's own memory area. The
sd-loader recognizes the loaded application and executes it at the next
application start. After loading, the graphical user interface triggers
a restart of the *Mii Maker* application and thus initiates the
execution of the homebrew application. A reopening of the Mii Maker
application starts the graphical user interface of the *Homebrew
Launcher*.

Own homebrew applications can be created using the devkitPPC development
environment, which is part of the devkitPro development
environment. With the *Homebrew Launcher*, it is possible to execute
homebrew applications in the execution file format of *Wii U* (.rpx), as
well as statically linked ELF files.

The ELF files, statically linked, must use an address as the start address,
which is located in the own memory area. Since the memory is limited,
the applications must not be larger than 6.4 MiB. To be
able to use the official system libraries, the *Homebrew Launcher*
provides the necessary addresses of the functions "OSDynLoad\_Acquire"
and "OSDynLoad\_FindExport". These can be read via fixed addresses in
the memory. With them, it is possible to load system libraries
("OSDynLoad\_Acquire") and get the addresses of their functions
("OSDynLoad\_FindExport"). The homebrew application is copied directly
to the address it is linked to and can be executed from there. By
statically linking, it is not possible to run multiple homebrew
applications at the same time because they would overwrite each other.
Since the code is position-dependent, it cannot be loaded to another
address.

In addition, with the *Homebrew Launcher*, it is possible to run homebrew
applications in the ".rpx" file format. The ".rpx" files offer the
possibility to link directly dynamically against system libraries. In
general, the loader of the Cafe OS is responsible for loading and
linking ".rpx" and ".rpl" files. It receives the data from the *IOSU*
after the integrity has been checked there. The *Homebrew Launcher* also
uses the loader to load the ".rpx" files. Because the integrity of the
files has already been checked in the *IOSU*, the received data can be
manipulated. In concrete terms, the loader is manipulated in such a way
that the data is used from its own memory area and thus the homebrew
applications are loaded. The maximum size of the ".rpx" file depends on
the size of the own memory area. The development environment/SDK
[wut](https://github.com/decaf-emu/wut) can be used to create your own applications in the ".rpx" file
format.

### Persistent code execution

In addition to the browser exploit as the entry point, further entry
points are possible. To achieve code execution, a vulnerability in the Nintendo DS
emulator is exploited when loading games. This exploit is known as
[haxchi](https://github.com/FIX94/haxchi).

In contrast to the browser exploit, which requires a connection to a
network to load a website, haxchi can be used completely offline after
installation and thus independently. However, control over the system is
already necessary for the installation, since the application files must
be modified. Therefore, haxchi cannot therefore be used as an initial entry point
into the system.

When the application is started, a game to be emulated is loaded from
the "/content" folder. The integrity of the files from this folder is
only checked at the time of installation, but not afterwards. If this
game is replaced with a modified version that uses an error in
interpreting the file, code execution is possible. From there, the
*Homebrew Launcher* can be temporarily installed and used, similar to
the browser exploit.

A system file (system.xml) defines which application is executed when the console is
started. If exploits can be used to control the *IOSU*, this file can be
modified to run an application with haxchi exploit instead of the system menu. 
This allows it to get code execution after starting the console with any manual user input. This
is also known as cold-boot haxchi or cbhc. This, however, comes with some risks. 
If the system can't boot the set title, the console will refuse to start.

### Manipulation of the Cafe OS

With the help of a kernel exploit it is possible to manipulate the
entire memory over which the PowerPC processor has control. This can be
used to modify and extend the code of the loaded system libraries.

This is partly done by the homebrew applications themselves to
manipulate the behavior of the Cafe OS. For example, there are homebrew
applications that redirect file accesses from the internal memory to the
SD card ([SD Cafiine](https://github.com/Maschell/SDCafiine)) or manipulate controller inputs ([HID to VPAD](https://github.com/Maschell/hid_to_vpad)).

Alternatively, the [*Wii U Plugin System*](https://github.com/Maschell/WiiUPluginSystem) application can be used to
load plugins. This is an application in ELF file format that is executed
via the *Homebrew Launcher*. The *Wii U Plugin System* allows the
simultaneous use of multiple plugins loaded from the SD card. The
plugins are normal ELF files that contain additional information in
their own ELF sections.

Plugins can be used to overwrite the functions of the system libraries
to modify their implementation. The first command of the functions is
overwritten with a jump into its own implementation. In addition,
plugins can be used to execute code in the background parallel to the
actual operation of the system. Furthermore, plugins can define
functions that are called by the plugin system during certain events of
the operating system. This could be, for example, starting or stopping
an application.

### Manipulation of the IOSU

To manipulate the *IOSU*, appropriate exploits are necessary to
gain control over the *IOSU*. A customized IOSU makes it possible, among
other things, to read cryptographic keys from the console, gain full
control over the console's file system, and temporarily disable digital
signature checks.

By disabling digital signature verification, it is possible to install
incorrectly signed content on the system. However, this content can only
be used as long as signature verification is disabled. As soon as the user 
restart the console, the titles can't be started anymore until the user 
uses the customized IOSU again.
Furthermore, it is possible to manipulate the rights of the running application that are
managed and implemented in the *IOSU*. This means, for example, that
access to the SD card can be obtained in any application.

At the time of this blog post, several implementations of
*IOSU* exploits and similar *IOSU* modifications are publicly available.
This combination of *IOSU* exploits and *IOSU* modifications is
typically called custom firmware (cfw). The most common custom firmware
implementations are [mocha](https://github.com/dimok789/mocha) and a custom firmware integrated in
haxchi.

*IOSU* modifications are implemented by either restarting the system
with a previously modified fw.img or by modifying the existing fw.img at
runtime. In theory, it is also possible to use a completely separate
fw.img, for example, to start Linux as an independent operating system using 
[linux-wiiu](https://gitlab.com/linux-wiiu/linux-loader).

### Current use of the solutions at a glance

All existing solutions presented in this subsection can be used without
permanently modifying the console. The browser exploit can be used to
launch the *Homebrew Launcher* and the Homebrew application. To
get control over the *IOSU* and thus the whole system, a custom
firmware, for example in the form of mocha, has to be executed
additionally.

By modifying the internal memory of the console, an application
compatible with haxchi can be permanently used as an entry point. 
To be able to perform these modifications, however, control over
the *IOSU* is necessary, which is achieved via the browser exploit and
an *IOSU* exploit. The *Homebrew Launcher* can be started once via the
browser exploit and haxchi can be installed via its own installer. An
*IOSU* exploit is integrated directly in the installer. After powering
on the console, the *Homebrew Launcher* and/or the Custom Firmware
integrated into haxchi can be used via this application by starting 
the haxchi application.

By modifying the system.xml in the internal memory of the console the
start of the console can be redirected to an application with haxchi.
Then the Homebrew Launcher or a custom firmware can be used immediately
after switching on the console.

## Meeting the requirements

With the existing solutions presented in this section, **requirement 1**
to run own software is fulfilled. Using the *Homebrew Launcher*, it is
possible to run homebrew applications in both ELF and ".rpx" file
formats. However, the size of the application is limited by the size
that can be defined as its own memory area. In addition, for loading the
".rpx" files, the loader of the Cafe OS is modified to replace the data
received from the *IOSU* with its own. With *IOSU* control, the loading
of ".rpx" files can be modified on the *IOSU* side, eliminating the need
to modify the loader.

**Requirement 2**, the use of the existing operating system, is also
fulfilled. The system can be used as usual. The manipulation of the
existing operating system requested in **requirement 3** is possible via
a custom firmware and the *Wii U* plugin system. *IOSU* changes are
possible with the Custom Firmware and Cafe OS changes with the *Wii U Plugin System*.
In general, code execution in the background by plug-ins
as required in **requirement 4** is possible, but not in combination
with homebrew applications. Because the *Wii U Plugin System* itself is
a homebrew application, plugins cannot be used simultaneously with other
homebrew applications.

**Requirement 5** requires that the homebrew environment be available
after the console starts. By using cold boot haxchi it is possible to
run a custom firmware or the *Homebrew Launcher* directly after starting
the console. However, it is not possible to use the *Wii U Plugin System* 
directly, which means that the requirements are only fulfilled
to a limited extent by the currently existing solutions.

**Requirements 6 and 7** are again met. It requires the homebrew
environment to be up to date with the latest version of the operating
system and without hardware modifications. The existing solutions only
use software exploits that can be used on the latest version of the
operating system.

One problem of existing solutions is maintainability. **Requirement 8**
requires that the homebrew environment be easy to maintain and update.
This is not always the case. The entry points use exploits with static
payloads. For example, the installer of the *Homebrew Launcher* is
integrated into the exploits that are used as entry points. Furthermore,
the installer is responsible for the complete setup of the homebrew
environment. To update the installer or sd-loader of the *Homebrew Launcher*,
all exploits that can start the *Homebrew Launcher* must be
updated. Exploits like haxchi would have to be completely reinstalled. A
centralized update is not possible.

There is no clear separation between the actual homebrew applications,
exploits and utility applications. The plugin system is implemented as a
homebrew application, although it is itself responsible for loading and
executing plugins. This prevents the simultaneous use of plugins and
homebrew applications. In addition, custom firmware is also
implemented as homebrew applications that are started by the user. An
application cannot rely on a custom firmware already running.
Applications that need control over the *IOSU* have partially integrated
the *IOSU* exploit or even a complete custom firmware into the existing
solutions. For example, an *IOSU* exploit is integrated with the haxchi
installer to allow access to the entire file system.

# Next blog entry
The next blog entry will propose a concept for a homebrew environment that meets the specified requirements.