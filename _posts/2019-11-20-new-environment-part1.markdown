---
layout: post
title:  "Creating a homebrew environment on the Wii U - Part 1"
date:   2019-11-20 18:05:00 +0100
categories: homebrew
redirect_from:
  - /bugs/2019/11/20/new-environment-part1
  - /bugs/2019/11/20/new-environment-part1.html
---

Some time ago, I went on a journey to create a new homebrew environment for the Wii U. 
An environment where homebrew applications can be run easily and on Wii U.
It's been possible to run your own software on Wii U since 2015, but I wasn't satisfied with the existing solutions. 
Some solutions, such as the Homebrew Launcher, date back to a time when no public IOSU exploit was known. 
In order to run Homebrew, some compromises had to be made. 

Over time the whole thing became more and more confusing for the users:

- There are different cfws (mocha vs haxchi) with different features
- Some apps require a special cfw (e.g. mocha with sd patches)
- There are two different formats for hombrews (rpx vs elf)
- Simultaneous use of plugins and homebrew application is not possible
- Updating any existing exploit is not quite user friendly
- and much more...

In the following blog posts I want to give you an insight into how I tried to create my own homebrew environment and fix these existing problems. 

This first post will be about giving a brief overview of the Wii U.

# The Wii U

Technical details about the exact hardware and software structure are
not available directly from *Nintendo*. Accordingly, the following
information comes from third parties and has been determined by reverse
engineering. A great source for internals of the system is [this blogpost](https://fail0verflow.com/blog/2014/console-hacking-2013-omake/) 
by fail0verflow, [WiiUBrew Wiki](https://wiiubrew.org/wiki/Main_Page) and the [decaf-emu](https://github.com/decaf-emu/decaf-emu).

# Operating System

The operating system of the *Wii U* is an own implementation of Nintendo
and is not based on an already existing operating system (like linux).
Exploits of similar systems cannot be taken over. On
the other hand, the probability of overlooking implementation errors
that can be exploited by third parties increases. Often there is a
cat-and-mouse game between the "hackers" and the manufacturers, in which
new exploits are found again and again and then fixed by Nintendo. 
Over time, this increases the security of the system.
It is only possible to run correctly digitally signed software. 
There is no possibility to officially run own code ("homebrew").

The *Wii U* operating system is split between the PowerPC and ARM
processors. The so-called *Cafe OS* runs on the PowerPC processor,
while the *IOSU* runs on the ARM processor. These are described in
detail below:

**IOSU**: The *IOSU* is based on a microkernel architecture and is
responsible for the boot process and all hardware accesses. It is a
continued development of the IOS, which runs on the ARM processor of the
Wii. In addition, it enforces the *Wii U* security guidelines. All
digital signatures are verified by the *IOSU*. The *Cafe OS* can
communicate with the *IOSU* via an IPC-interface.

**Cafe OS**: The PowerPC processor runs the *Cafe OS* in which the
applications are executed. The *Cafe OS* itself consists of several
components. On the one hand there is the kernel, which manages the
processor. The kernel runs in "supervisor mode" and manages the running
application, the mapping of the physical memory to the virtual memory
and is responsable for the process isolation. In addition, communication with
the *IOSU* is handled by the kernel. On the other hand, there is the
loader, which is responsible for loading and dynamically linking the
executable files. In addition to the actual application, the loader also
loads software libraries that are themselves part of the *Cafe OS*. The
execution files ".rpx" and software libraries ".rpl" on the *Wii U* 
based on the ELF fileformat with some additions. The individual ELF 
sections are compressed and the imports and exports are defined in separate sections.

Applications run on the PowerPC processor with user privileges and thus
with restrictions - including limited access to memory. For example, it
is implemented that regions in memory cannot be written to and executed
by the application at the same time. Among other things, this concept
prevents trivial code execution, which becomes possible if the stack of
a thread can be manipulated via a buffer overflow.

This division of tasks between the processors has the side effect that
control over the PowerPC processor where the games are running 
does not allow control over the entire
system. If an error in an application is exploited and the ability to
execute code is obtained, it is still not possible to read sensitive
information from the *IOSU*, such as cryptographic keys used to decrypt
critical parts of the system. A separate MPU[^7] ensures that the memory of the *IOSU* cannot be read despite
control over the PowerPC.

Applications are executed either from the internal memory or from an
optical medium. The files of an application are divided into three
subdirectories, "/code", "/content" and "/meta", which are described in
more detail below.

**/code**: The "/code" folder contains execution files, additional
libraries and configuration files that configure the memory or the
permissions of the application, for example. The executable files are
loaded by the loader, i.e. directly on the PowerPC processor, and linked
dynamically, with the data coming from the *IOSU*. The remaining files
are processed directly by the *IOSU*. The integrity of these files is
ensured by the *IOSU* each time the application is started.

**/meta**: The "/meta" folder contains the meta information of the
application, such as the electronic manual, graphics displayed at
startup, and another configuration file. The "meta.xml" file stores,
for example, the title of the application in the various
languages.

**/content**: The actual data used by the application is stored in the
"/content" folder.

As soon as the user opens the "Home Menu" via the controller, the
current application will then only be running in the background. From
there, the application can be closed or additional applications such as
the browser or the friends list can be launched. While a application is running
in background it's limited to use only one of the CPU-Cores.

# Boot process / Chain of Trust

With the *Wii U* the boot process takes place in several stages. 
The *Wii U* implements the
concept of the *chain of trust*. If the integrity of one stage is
guaranteed, the integrity of the next stage can be guaranteed. If the
Chain of Trust is broken at one point, the integrity remains intact up
to the previous stage.

The boot process of the *Wii U* starts with the first stage of the
bootloader **boot0**. Boot0 is located in a ROM within the ARM
processor, embedded in hardware. This means that it can only be read,
but not changed, and thus provides the basis for the *chain of trust*,
i.e. the so-called "Trust Anchor". The manufacturer must ensure that
this level does not contain any vulnerabilities, as these cannot be
patched by software. The task of this stage is to read the next stage of
the bootloader boot1 from the internal memory, decrypt it and then
verify its digital signature. On success, the boot1 stage is executed. A
pseudo code of this step can be found [here](https://wiiubrew.org/wiki/Boot0).

The **boot1** stage is not implemented in hardware, so it can be updated by
the manufacturer, but it is digitally signed. The digital signature
ensures the authenticity and integrity of the level. A suitable digital
signature from a third party cannot be created without the private key.
This concept is consistently implemented for all further stages. At this
stage, the hardware is initialized and then a further stage in the form
of a fw.img file is loaded from the internal memory, decrypted and its
digital signature verified. The fw.img file contains the *IOSU*, the
operating system running on the ARM processor. It consists of a simple
ELF loader and the actual *IOSU* modules. This ELF loader is executed at
the end of the boot1 stage. A pseudo code of level boot1 can be found
under [here](https://wiiubrew.org/wiki/Boot1).

The **ELF loader of the *IOSU*** is responsible for loading the *IOSU*
kernel. This is responsible for loading the remaining *IOSU* (user)
modules. It also loads the kernel for the PowerPC processor, which is
part of the *Cafe OS*. This is loaded from the internal memory of the
kernel.img file. The ELF loader decrypts the kernel and verifies its
digital signature.

The **PowerPC kernel** is responsible for initializing the PowerPC processor
and loads the loader from internal memory. This is also decrypted and
the integrity and authenticity is checked using the digital signature.
System libraries and applications can then be loaded via the loader.

A *chain of trust* is given. Each stage checks the next one, while the
initial stage is unchangeable and trustworthy. This extends all the way
to the application that the user sees at the end. The execution of an
own firmware is basically not possible at startup, even if the
corresponding files are exchanged. As a result, *Nintendo* can be the
only company that releases firmware. In order to gain control over the
system, the *chain of trust* must be broken at any point.

# How to gain control over the system

The sooner the *chain of trust* can be broken, the sooner control over
the system can be gained. Earlier control also provides more control
over the system because less code was running that cannot be controlled.
The integrity of all potentially subsequent stages is no longer assured.
If the *chain of trust* is broken at a later point in time, the
integrity of all previous levels remains intact. In order to be able to
manipulate the preceding levels, vulnerabilities in them must be
exploited.

It is desirable to gain control over the system already in the first
stages. This is possible on some consoles like the [*Nintendo Wii*](https://wiibrew.org/wiki/BootMii)
, the [*Nintendo 3DS*](https://www.3dbrew.org/wiki/3DS_System_Flaws#Boot_ROM) or
the [*Nintendo Switch*](https://switchbrew.org/wiki/Switch_System_Flaws#Hardware). On the *Wii
U*, at bootime this was only possible with a ["glitching"-setup](http://wiiubrew.org/wiki/Wii_U_System_Flaws#boot0).
In fact, there is also an exploit that
allows [code execution in the boot1 stage](http://hexkyz.blogspot.com/2018/01/anatomy-of-wii-u-end.html), but the console must have been
completely started and taken over. When the console is started, the
*boot0* level stores a data structure in the main memory. If a restart
is initiated while the console is running, the boot process starts at
the boot1 stage in which the existing data structure is read. Thus it
can be used that the main memory is not emptied during a restart of the
console, whereby a manipulation of the data structure is possible. An
error when verifying the values can be used to gain control over the
boot1 stage. A solution that allows control during
the boot process without modification of the hardware and immediately
after power-on does not yet exist.

Instead, it is necessary to gain control of the operating system in
another way. Vulnerabilities in applications can be exploited to execute
custom code. If you succeed in gaining additional control over the
kernel of the PowerPC processor, you have complete control over the
processor and the *Cafe OS* running on it. For control over the *IOSU*
and thus the ARM processor, an *IOSU* module and ultimately the *IOSU*
kernel would have to be taken over via the IPC interface by exploiting
security vulnerabilities. It is necessary that the described steps are
performed each time the console is switched off. Persistent changes to
the operating system would violate the integrity of the data, which
would prevent the console from being started.

All cryptographic keys on the console are known publicly and can be read
with publicly available applications such as [hexFW](https://github.com/hexkyz/hexFW). This makes it
possible to decrypt all console contents and analyze them for bugs. In
order to be able to digitally sign and use your own content, however,
the appropriate private key is required. This key is not known and will
probably never reach the public. A large part of the system has already
been analyzed, however, and corresponding information can be found on
the Internet (for example in [WiiUBrew-Wiki](http://wiiubrew.org/)). Exploits are also
publicly available, and already allow full control of the system
(for example using the [homberew launcher](https://github.com/dimok789/homebrew_launcher) in combination with [mocha](https://github.com/Maschell/mocha/) or by running [hexFW](https://github.com/hexkyz/hexFW)). Information
about the initial obtaining of full control and the reading out of
corresponding keys via the *Wii U* can be found for example [here](https://www.youtube.com/watch?v=oss_dwj-IkE).

# Next blog entry
The next entry will discuss the requirements that will be used for creating the environment. 
It will also discuss which parts of the existing solutions are bad and need to be improved, and which parts can reused.