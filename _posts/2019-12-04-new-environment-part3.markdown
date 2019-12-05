---
layout: post
title:  "Creating a homebrew environment on the Wii U - Part 3"
date:   2019-12-04 18:00:00 +0100
categories: homebrew
---

In this post a homebrew environment is to be designed, which fulfills
all requirements from [part 2]({{ site.baseurl }}{% link _posts/2019-11-27-new-environment-part2.markdown %}). 
At the same time the knowledge about already existing solutions shall be considered. The
extent to which these solutions can be reused and which components need
to be revised or newly developed will be discussed in this post.

Existing solutions have so far been designed and developed independently 
of each other. On the one hand they have
been developed one by one, on the other hand the solutions come
from different, independed developers. No overall objective was followed. This leads
to some problems, whereby the defined requirements are not fulfilled.
These have already been explained in the [last blog entry]({{ site.baseurl }}{% link _posts/2019-11-27-new-environment-part2.markdown %}).

The following goals can be derived for the conception:

-   It should be possible to execute homebrew applications.
-   At the same time it should be possible to use plugins that can
    modify the Cafe OS and execute code in the background.
-   IOSU modifications should be available in the homebrew environment
    so that they can be expected from applications.
-   Maintainability and easy updating should be possible (requirement
    8), with as little effort as possible for the end user.
-   The requirements already fully met by the existing solutions should
    continue to be met.

A concept that achieves these goals is developed in the following step.

# Inital code execution

An initial code execution must be
enabled so that the setup of the execution environment and thus the
execution of homebrew applications is possible. Since a separate code
execution is not intended, it is necessary to exploit a bug in an
existing application. In general, applications in which data enters
the system or is modified by the user can be considered. This content
could, for example, be savegames, QR codes, websites or other media for a
media player.

On Wii U, it is unfortunately not possible to manipulate savegames in
order to take advantage of possible bugs when reading the savegames.
Wii U content can only be copied to an external medium (USB device) if it's formatted in a [special file system](https://github.com/koolkdev/wfslib).
In addition, the medium is also encrypted.
The key used for encryption is unique for each console, which means that
no content can be transferred from an already modified console.
Therefore, it is not possible to use modified savegames on a console
without having the corresponding keys. To obtain this key other exploits would be needed. 
Because of this, savegames exploit are not feasible as primary entry exploits.

Another possible entry point would be the integrated web browser. The
following reasons speak for the use of the browser as an entry point:

-   The web browser is pre-installed on each console.
-   The content of a website can be freely crafted.
-   Webkit is a well-known browser engine.
-   Through irregular updates, security vulnerabilities are fixed with
    delay on the Wii U.
-   An access to a memory area which is writable and executable at the
    same time is given. (JIT-area, internally referred as codegen area)

A website with arbitrary content can be created and viewed via the
browser, resulting in a huge attack vector. Due to the complexity of
today's browsers, security vulnerabilities might occur. Because a known
browser engine is used, exploits are already known and can potentially
be used. In addition, consoles often do not receive regular browser
updates. The [browser versions](https://wiiubrew.org/wiki/Internet_Browser#Known_User_Agent_Strings) used are often several years old. 
Information about fixed bugs [can be found](https://www.cvedetails.com/product/10007/Apple-Webkit.html?vendor_id=49)
on the Internet. The probability that an already
existing browser exploit can be used is correspondingly high.

In addition, modern browsers use a JIT compiler to efficiently execute
JavaScript. The executed JavaScript code is converted to native code at
runtime and then executed. This requires a part in memory that is
writable and executable at the same time. Typically, applications do not
have access to this area. In the case of the browser, however, this is
necessary for efficient execution. In the following, this area in memory
is referred to as the "JIT area".

The memory available for the JIT(just in time) compiler in the case of the Wii U
[is 32 KiB big](https://wiiubrew.org/wiki/Cafe_OS#Virtual_Memory_Map). By exploiting the browser with a
[ROP](https://en.wikipedia.org/wiki/Return-oriented_programming)(**R**eturn **O**riented **P**rogramming) chain, you can use this memory for your own code. 
For a ROP chain a manipulation of the stack is required. The stack is modified in
such a way that the return addresses execute parts of the existing code
and thus achieve a desired behavior. Using the ROP chain it would be possible
to copy code into the JIT area, where it can be executed, resulting in a true
arbitrary code execution.

This execution is not only limited by the size of the JIT area. At the
same time, code execution is limited to the browser context. Any browser
restrictions apply equally to the own code.

It is therefore desirable to achieve a "better" code execution,
with which it is possible to execute more code in a cleaner context. 
With the exception of the JIT area, applications do not have areas in memory that can be
written and executed at the same time. This is implemented via the
MMU(**M**emory **M**apping **Unit**), which maps the physical memory to virtual memory with
corresponding permissions. This mapping as well as the permissions can
only be changed with (PPC) kernel privileges. This requires another exploit to
make the required modifications. In the following, this exploit is
referred to as the "kernel exploit". Alternatively, it is also possible to
temporarily disable the MMU. Meanwhile, the memory can be accessed via
the physical addresses. This makes it possible to copy arbitrary code
into executable areas of the memory. The physical addresses can be
obtained for the respective virtual addresses via functions of the
system libraries (OSEffectiveToPhysical).

In order to increase maintainability, it makes sense to load potentially
changing code parts from an external source. This also has the advantage
that an abstraction takes place. Each potential entry point should only
contain as much code as is needed to allow a unified payload to be
loaded from external. In the case of an update, only this single payload
would have to be updated, not the actual exploits or entry points. 
With current solutions for example haxchi and the browser exploit needs to be updated independently.

Loading a secondary payload would to be possible to load from the SD card or via the network. 
When loading over the network, network access is required. In the case of a browser
exploit, this is required anyway; however, other exploits, such as
haxchi, work completely offline and independed. Therefore the payload should be stored
on the SD card. There it can be easily replaced by the user. This,
however, creates a dependency on the SD card.

In order to load the payload from the SD card, access to the SD card is
necessary. Like most applications on the Wii U, the browser does not
have access to the SD card. The pre-installed application *Mii Maker*
offers the option to store images on the SD card, so it has access to
the SD card. To access the SD card, you need to switch to the *Mii Maker* 
application. When the applications are switched, the memory
available for the application is cleared. At the same time, the change
also eliminates access to the JIT area and the possibility of code
execution. To ensure that code execution is still possible, a memory
area is required that can be written arbitrarily and can also be
executed. In addition, the content of this memory area must survive the
change of the currently running application in order to be able to store
a payload there.

Such a memory area can be set up using a kernel exploit. A payload must
then be placed in this memory area, which is executed in the context of
the *Mii Maker* application. It is then necessary to manipulate the Cafe
OS in such a way that the *Mii Maker* application executes this payload
when it is started. For example, the call of the "main" function of the
application can be replaced by the call of the payload. The necessary
parts of the Cafe OS can be modified thanks to the kernel exploit. In
the following this payload is called *"main hook"-payload*.

This *"main hook"-payload* will be executed in the context of the *Mii Maker*  as soon as the Mii Maker is opened.
It will be used to load another payload from the SD card.
This payload will be called *payload.elf* in the future.

<center><img src="/res/payload_elf_chain.png" width="450"></center>

The figure
aboce shows the procedure for loading and
executing *payload.elf*, which can be summarized as follows:

1.  Obtaining ROP chain execution using an exploit.
2.  Using the ROP chain, copy a payload to the JIT area and execute it.
3.  Use a PowerPC kernel exploit to copy another payload into an area
    that survives a switch to the *Mii Maker* application.
4.  The Cafe OS must be manipulated so that this payload is executed
    after a change of application.
5.  Switch to the *Mii Maker* application, which allows access to the SD
    card.
6.  The payload now has access to the SD card and makes it possible to
    load another payload from the SD card.

On one hand the steps 1 and 2 would need to be created specifically for each application. 
ON th eother hand the steps 3-6 would be completly independent of the specific exploit and could be re-used without any changes.

# Code execution is possible

At this point it is possible to load and execute a payload from the SD
card. Theoretically from this point on there is sufficient code
execution to execute any code. However, homebrew applications should
also be executable in the ".rpx" file format, which is not yet possible
at this time. It's also not possible to run code in the background.

In general, it makes sense that future homebrew applications should only
use the ".rpx" file format. This is the native file format for
executable files on the console and offers the possibility to use the
system libraries directly. All homebrew applications that require a
deeper manipulation of the system (such as replacing system functions) can be implemented as plugins for the
Wii U Plugin System.

In the existing solutions, the ".rpx" files were executed by
manipulating the original [Cafe OS loader](http://wiiubrew.org/wiki/Loader). 
This was done because no control over the IOSU was possible yet. However, it is now possible to simplify this loading
via IOSU modifications. A [proof of concept](https://github.com/QuarkTheAwesome/mocha/commit/8e56a1b2ce15f64fdb016a0dfe197c38be9b453e) from the developer
["QuarkTheAwesome"](https://heyquark.com/aboutme/) shows how file paths can be redirected to the SD card
when loading ".rpx" files. In addition, the
integrity checks of the ".rpx" files must be deactivated so that own
files can be loaded.

This approach has the advantage that, except for the manipulation of the
path, the loading is handled by the operating system. In addition,
it is no longer necessary to store the file in a separate memory area,
which may be limited in size.

So that homebrew applications can be executed controlled from the SD
card, a homebrew launcher should be implemented analogous to the
existing one. Applications located on the SD card should be able to be
started via this launcher. The launcher will consist of a graphical user
interface which controls the redirection of the ".rpx" files. This
Homebrew Launcher can itself be implemented as a ".rpx" file and
executed via the redirections. To be able to start the Homebrew
Launcher, it should be opened when starting an application. This
requires a fixed rule in the ".rpx" redirection which exchanges the path
to this application with the path to the "Homebrew Launcher" which is
located on the SD card. Which application is exchanged is not relevant.
However, it is a good idea to select an application that is rarely used (for example the Health and Safety application).

In order to load ".rpx" files, an IOSU exploit is required to manipulate
the operating system to allows redirections. This IOSU exploit can be implemented in the form
of a *payload.elf*, which is loaded from the SD card from PPC userland exploits.

The IOSU exploit can either be used to load a modified fw.img from the SD
card, or to manipulate the IOSU in memory on-the-fly. After a restart, the
desired changes take effect. The problem with using a full custom fw.img is 
that it may contain copyrighted code and is not that easy to share.


In view of a possible future vulnerability in the bootrom, it would also be thinkable to create a own fw.img which has not this problem.
An exploit of the bootroom would allow the execution of a fw.img directly at the start of the console.
If it is possible to boot directly into a own fw.img, this can serve as an early code execution and then modify and execute the original fw.img "on-the-fly".

Such a fw.img could also be executed via usual IOSU exploits, but
without a corresponding vulnerability in the bootrom it does not bring
any advantages. For this reason, the IOSU modification should be done
via the *payload.elf* loaded from the SD card. All possibilities to
restart into the modified IOSU are shown in the following figure:

<center><img src="/res/fw_img.png" width="550"></center>

After a restart into the modified IOSU, the corresponding IOSU
modifications, and thus also the ".rpx" redirection, are available. A
fixed redirection ensures that the start of a previously defined
application loads a ".rpx" file, in this case a Homebrew Launcher, from
the SD card. This Homebrew Launcher should enable the user to load any
homebrew application from the SD card. This requires a mechanism to
control the ".rpx" redirection. The IPC(**I**nter-**p**rocess **c**ommunication) interface for communication
between the Cafe OS and IOSU will be extended by a new IOCTL. The Homebrew
Launcher also needs access to the SD card in order to be able to show
the user a corresponding selection of possible homebrew applications. A
IOSU modification has to bypass the restriction that only
certain applications have access to the SD card.

Theoretically it would be possible to execute the IOSU exploit directly
in the browser exploit and restart the operating system with the desired
changes. Switching to the *Mii Maker* application and thus a kernel
exploit would no longer be necessary. A sufficient code execution is
given by the JIT area. However, the abstraction of the entry points and
thus the central payload on the SD card is lost. If the IOSU exploit is
modified or the desired IOSU modifications change, all entry points
would have to be updated (independently). Only a own fw.img, which
is used to start the original fw.img with on-the-fly modification, would benefit speed wise. 
A switch to the Mii Maker application wouldn't be needed anymore. However,
this would lose the flexibility offered by a central payload from the SD
card. It would only be possible to execute a fw.img file, but no longer
generic code, from the SD card.

The following figure summarizes the current state starting from a
*payload.elf*. Homebrew applications can be loaded in the form of ".rpx"
files after a restart via a homebrew launcher. For this purpose, a
*payload.elf *is loaded from the SD card, which performs an IOSU exploit
and any necessary IOSU modifications. After a restart these
modifications take effect and the Homebrew Launcher can be started by
starting a previously defined application. The Homebrew Launcher can be
used to start the individual Homebrew applications.

<center><img src="/res/hbl.png" width="450"></center>

At the same time it's still possible to create a *payload.elf* implementation that 
recreates the old solution to provide backwards compatibility.

# Code execution after restart

After rebooting into a modified IOSU, the Cafe OS will also be reloaded,
causing all temporary changes on the PPC side to be lost. Theoretically, it would be
possible to reboot into a modified Cafe OS, but this would add a
dependency on the IOSU exploit based on the implementation.

Instead, the privileges should be regained independently by using a
kernel exploit. Should only an IOSU exploit but no kernel exploit be
possible in a future operating system version, manipulation of the
PowerPC kernel from the IOSU would be an option.

The ".rpx" redirection can be used to get code execution after a
restart. After the restart, the system menu is loaded first. It is a
reasonable idea to redirect the loading of the corresponding ".rpx" file
to the SD card once so that code can be executed immediately after the
operating system has been restarted. In the following, this ".rpx" file
is referred to as *root.rpx*.

With the help of the code execution, the operating system can be set up
on the PowerPC processor. Full control over the PowerPC processor and
thus the Cafe OS can be obtained via a kernel exploit.

In order to keep code execution, the Cafe OS has to be modified. Again
the invocation of the "main" function is replaced by a jump to the own
code. For this payload, a separate memory area must be defined,
analogous to the original homebrew launcher in which the payload is
stored. The payload can be loaded from the SD card and is then called
each time the application is opened. This allows code execution parallel
to the actual homebrew applications. In the following this payload is
called *hook\_payload.elf*. An overview is shown in the following figure:

<center><img src="/res/root_rpx.png" width="350" align="middle"></center>

In addition, the homebrew environment on the PowerPC processor can be
further set up and configured.

# Manipulation of the Cafe OS

Each time an application is started, *hook\_payload.elf* is executed,
which provides ongoing code execution. This is also the case if homebrew
applications are started via the homebrew launcher.

Among other things, it should be possible to manipulate the system
libraries of the Cafe OS. To do this, the Wii U plugin system can be
reused. This makes it possible to load plugins from the SD card which
can manipulate the functions of the system libraries.

For this the plugin system has to be updated. So far it was implemented
as an application for the Homebrew Launcher. Now the plugin system
should run independently from the Homebrew Launcher to allow
simultaneous use. For this purpose the plugin system must be executable
without the previous environment of the Homebrew Launcher. On the other
hand, the actual business logic and the graphical interface, which is
used to activate the plugins, must be separated. The business logic is
to be integrated into *hook\_payload.elf* and a graphical user interface
is to be implemented as a homebrew application (.rpx).

With the help of the integrated plugin system in *hook\_payload.elf* a
code execution in the background as well as the manipulation of the
functions of system libraries by plugins is possible. The functions can
be manipulated by directly replacing the first instruction of a function in memory in such a
way that own code is executed instead. If the overwritten instruction is
remembered, the original function can also be executed. System events
can be propagated to the plug-ins by specifically hook into
certain functions of the system libraries. By creating threads, code can be executed in the
background for use. Due to the direct code execution after starting the
homebrew environment, it is also possible to execute plugins straight
away after booting into the homebrew environment.

A mechanism for communication with the business logic of the plugin
system is required so that a graphical interface in the form of a
homebrew application can control the plugins. For this reason it is
necessary to create a kind of IPC interface between *hook\_payload.elf*
and homebrew applications.

It would be optimal if these background operations did not take away any
resources, such as memory, from the actually running application. If
enough unused memory is available, it can be mapped as virtual memory
using the kernel exploit. It may be possible that this memory can be
used as heap for plugins and other background processes that run
simultaneously to the normal system operations.

In summary, the homebrew environment can be seen in the follwing figure:

<center><img src="/res/umgebung.png" width="550" align="middle"></center>

# Full control over the system

So far, the IOSU exploit has been used to redirect the ".rpx" files.
However, other modifications can be implemented using the exploit.

Access to the file system is controlled by the IOSU. Each application
only has access to its own files, so system files are not accessible. In
order to gain access to the entire file system, the IOSU must be
modified accordingly.

Furthermore, (console-specific) cryptographic keys can be read. These
can be used, for example, to decrypt content or external media.

Similar to the communication between homebrew applications and the
*hook\_payload.elf*, homebrew applications must be able to communicate
with the IOSU in order to benefit from these modifications. In this
case, the existing IPC interface can be used for communication between
the ARM and PowerPC processors. However, it is necessary to register a
separate endpoint with its own functions (wupserver).

In addition, the IOSU exploit can be used to access the full hardware.
Among other things, the internal storage could be accessed directly,
allowing a backup or recovery. These functionalities are not necessary
in the planned execution environment, but can be implemented in a
separate *payload.elf* if required. Alternatively a fw.img of the already existing
hexFW mentioned in previous blog posts can be used. 

# Persistent code execution

All changes discussed so far are only temporary and disappear after
restarting or switching off the console. To be able to use the homebrew
environment again, it must be set up again via the browser (or any other entrypoint). The goal is
to achieve an (optional) persistent executon enrionment where the
browser exploit does not have to be executed manually.

The Wii U system files contain a configuration file that defines which
application should be executed after the console has been started ("system.xml"). It is
possible to manipulate this file using the IOSU exploit. If you select
an application that executes an exploit immediately, the console is
effectively started in the homebrew environment.

Unfortunately, the browser is not suitable for this because the web page
containing the exploit has to be called manually. In addition, this
would mean a dependency on a network connection, an independent use
would not be possible.

Complete access to the file system increases the attack vector for
exploits. Manipulating files used by applications is now possible. In
addition to the savegames that can now be manipulated, it is now
possible to manipulate any files from applications, since their
integrity is only checked during installation, not at runtime (this is sometimes referred as "contenthax").

It makes sense to use the haxchi exploit already introduced in the last blog entry.
This exploits a
security vulnerability in applications that have integrated a Nintendo
DS emulator. The ROM to be executed, i.e. the game, is located in the
"/content" folder, whose integrity is not checked at runtime. It is
therefore possible to exchange or modify this ROM. Due to a bug when
reading this ROM, a ROP chain execution is possible. Like the browser,
the Nintendo DS emulator also has access to the JIT area. The ROM is
read in directly after the application has been started and is therefore
suitable for persistent execution after the console has been started.
Similar to the browser exploit, the code execution can be used to load
the *payload.elf* from the SD card. The remaining procedure is identical
to the previous execution via the browser exploit, and thus starts the
homebrew environment. An update of the homebrew environment via
*payload.elf* has effects on a start via the browser exploit as well as
the haxchi exploit, resulting in a good maintainability.

It has to be considered that the original application, which executes
the haxchi exploit, can no longer be used. In addition, the memory of
the console is permanently changed and a compatible application must be
purchased, resulting in one-time costs.

Furthermore, it should be noted that a direct start in the haxchi
exploit, i.e. coldboot haxchi, also involves risks. The manipulation,
which application is executed when starting the console, is a deep
manipulation of the system. If the target application fails to start for
any reason, the entire console can no longer be used. This is also the
case if this application is accidentally deleted.

# Fulfillment of the goals

The goals defined at the beginning of the blog post can be achieved with
the presented concept.

With the help of ".rpx" redirections, which can be controlled via a
graphical user interface in the form of a new homebrew launcher, it is
possible to execute homebrew applications. The plugin system shall be
embedded in the *hook\_payload.elf* and a control via a homebrew
application shall be possible. This decoupling of the Plugin System from
the Homebrew Launcher allows a simultaneous use. Homebrew applications
can only be executed after the execution environment has been started.
Thus the applications can now expect that corresponding IOSU
modifications have been performed. All relevant payloads that could
potentially change are loaded from the SD card. This makes it easy to
maintain and update the individual components of the homebrew
environment.

In summary, this last figure shows an overview of the concepts presented in
this blog entry.

<center><img src="/res/gesamt.png" width="600" align="middle"></center>