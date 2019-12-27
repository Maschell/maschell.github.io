---
layout: post
title:  "Creating a homebrew environment on the Wii U - Part 4"
date:   2019-12-27 14:30:00 +0100
categories: homebrew
---

This blog entry is intended to discuss the implementation some of the concepts
developed in [part 3]({{ site.baseurl }}{% link _posts/2019-12-04-new-environment-part3.markdown %}).
It will focus on getting stable code execution in form of loading a `payload.elf` from the sd card.
The implementation discussed in this blog post has already been released back in [january 2019](https://gbatemp.net/threads/more-stable-webhack-for-5-5-2-5-5-3-5-5-4-released.528757/).

# Initial code execution

To achieve initial code execution, a bug in the browser can be
exploited. The necessary steps are then implemented with the goal of
loading a payload from the SD card.

However, the implementation of the necessary browser exploit does not
have to take place from scratch. The existing browser exploit with the
name [JSTypeHax](https://github.com/WiiUTest/JsTypeHax), which exploits the [CVE-2013-2857](https://nvd.nist.gov/vuln/detail/CVE-2013-2857)
vulnerability, can be used as the basis. 
It is exploiting heap-use-after-free
bug, that can be exploited via JavaScript code. It is taken advantage
of the fact that the browser still uses a **reference A** to **object A** after
freeing it. If an **object B** is allocated directly afterwards, the data is
located at the point in the memory where **object A** previously was. A
manipulation of the memory to which the **reference A** points is
effectively possible via **object B**. The data is stored in the memory
where **object A** was previously located. The trick is that **object B** uses a
different data structure. This makes it possible to effectively
manipulate parts of **object A** to which no access is given via **reference A**.
The data structure of **object B** can be chosen freely by using a different data
structure. The use of **reference A** may cause unexpected values to be
used. By a careful choosing these values, a manipulation of the stack,
and thus a ROP chain execution, is possible. Subsequently, a payload is
written to the JIT area via the ROP chain and executed.

The problem with the existing implementation is the success rate. Only
in a few cases does the code execution succeed. This is to be improved.
In addition, the loading of `payload.elf` from the SD card must also be added.

## Previous implementation

In the existing implementation, code execution is achieved through the
following (simplified) steps.

1.  Placing a ROP chain in memory by creating JavaScript objects
    containing the ROP chain.
2.  Similarly, the payload to be executed is placed in memory.
3.  A security vulnerability in the browser is exploited to allow
    manipulation of the stack.
4.  The manipulation of the stack executes the ROP chain placed in
    memory.
5.  Via the ROP chain, the payload is copied to the JIT area and then
    executed.

For this to work, addresses must be predicted at which the ROP chain and
the target payload are located in memory. It is not possible to use
JavaScript to store data at specific addresses in Wii U memory. It is
therefore not possible to predict exactly in advance where the data will
end up in memory.

In order to increase the chances that the corresponding data is really
at the predicted address, it is stored several times in succession in
the memory. This is also known as "heap-spraying". This increases the
probability that a predicted address will actually point to the target
data.

The problem here is that the target data is at the predicted address,
but not necessarily the beginning of the data. In order to avoid this, a
so-called [NOP-slide](https://en.wikipedia.org/wiki/NOP_slide) is appended in front of the payload. This
corresponds to a series of instructions that are interpreted by the
processor as a null operation. If, for example, 1000 null operations are
appended to machine code before the payload, it is sufficient to predict
one of the addresses of the 1000 null operations. The intended code is
executed afterwards.

<center><img src="/res/nop.png" width="400" align="middle"></center>

Starting at the predicted address, the memory is copied to the JIT area
and executed. This is shown schematically in the figure
above. The null operations at the beginning would
have no effect, but at some point the payload would be executed. In
general, the longer the NOP-slide, the higher the chance of success,
because the predicted address only has to hit a point in the NOP-slide.
However, the size of the NOP-slide is limited to 32 KiB, since the NOP
slide and payload must fit together in the JIT area.

Due to the preceding null-operations, the position of the code
potentially changes with each execution. This, however, creates the
problem that the payload cannot contain any position-dependent code. To
counteract this problem, another small payload is inserted between the
NOP slide and payload. This payload is position-independent and can
therefore be executed without problems. The task of this payload is to
determine the actual address of the payload to be executed.

For this reason, a special payload is created that consists of the
following parts:

1.  A preceding NOP slide with which the maximum 32 KiB is filled.
2.  A small, position-independent payload to determine the address of
    the payload to be executed.
3.  The size of the payload to be executed.
4.  The position-dependent payload to be executed.

This special payload is spread across the heap.

Via the ROP chain, 32 KiB are copied to the JIT area and executed,
starting from the predicted address of the memory. If an address located
in the NOP slide is successfully predicted, these null operations and
then the small position-independent payload are executed. This is shown in the figure below.

<center><img src="/res/old_nop1.png" width="400" align="middle"></center>

This payload knows its current execution position, the predicted
address, and its own size. This is sufficient to determine the correct
address of the payload to be executed. The payload to be executed is
then copied from the JIT area to the predicted address, this time
without the NOP slide and extra payload. This is shown in the figure below. 

<center><img src="/res/old_nop2.png" width="400" align="middle"></center>

The ROP chain can now again copy and execute the data from the now correctly predicted address into
the JIT area, whereby the position-dependent payload is executed
correctly. This is shown in the figure below.

<center><img src="/res/old_nop3.png" width="400" align="middle"></center>

Detailed technical notes and annotated code of this implementation can be found [here](https://gist.github.com/orboditilt/c4de45d284ba14bceef3d49b45312e52).

The previous implementation currently has two critical points at which
predictions must be made. If one of the predictions is not precise
enough, the execution of the exploit fails and the console crashes. 
The new position of the stack (containing the ROP-Chain) and the position payload copied to the JIT-areas have to be preditcted.
The prediction of the payload address is not always successful. 
Because the possible size of the NOP-slide is limited by the size of the JIT area, the success rate can't be increased "infinitely" by extending the NOP-slide.
The NOP-slide plus payload can only be a maximum of 32KiB. As a result, the
probability of predicting an address in the middle of the payload and
not before (inside the NOP-Slide) increases, resulting in a crash.

## New improved implementation

After some tests it was clear that the ROP-Execution is stable, 
but executing the payload fails most of the time.
To increase the success rate of the browser exploit, the loading of the
payload must be optimized.

Up to now, attempts have been made to increase the required accuracy by
prefixing a NOP slide, which, however, can only have a limited length.
This length is limited by the size of the JIT area. If a payload is
written directly to the JIT area via the ROP chain, which helps to
determine the position of the payload to be executed, the restriction
can be circumvented.

This results in a new procedure:

1.  Place a ROP chain in memory by creating corresponding JavaScript
    objects.
2.  Similarly, the payload to be executed is placed in memory, preceded
    by a unique value.
3.  A vulnerability in the browser is exploited to allow manipulation of
    the stack.
4.  The manipulation of the stack is performed by the ROP chain placed
    in memory.
5.  A small payload is written directly to memory to a specific address.
6.  This small payload is copied to the JIT area and then executed.
7.  Starting at a predefined address, this payload searches for the
    real payload to be executed using the unique value. If it's found,
    the payload copies it to a pre-defined address in memory.
8.  The ROP-Chain will then copy it from pre-defined address to the 
    JIT area, where it can be executed from.

This new procedure simplifies the prediction of addresses. The value
that precedes the payload should be selected so that it probably does
not occur elsewhere in the memory. This ensures that as soon as the
value is found in the memory, the payload is behind it. For a successful
execution, it is only necessary that the unique value and payload are
behind the start address of the search. Crashes of the exploits are very
rare now.

### ROP-Chain generation

The previous implementation generates the ROP chain within the
JavaScript code. In the improved implementation the existing ROP chain generator
[wiiuhaxx_common](https://github.com/yellows8/wiiuhaxx_common) will generate the ROP chain. This is implemented
in PHP and is already used in [browser exploits for earlier
operating system versions](https://github.com/yellows8/wiiu_browserhax_fright). The
wiiuhaxx_common ROP chain generator offers a dynamic generation, which
can be used by different exploits. It only uses gadgets in in system
libraries, which are loaded at any time and at the same location in
memory. Thus, the ROP chains created by wiiuhaxx\_common can be used system-wide, 
independent of the running application.

The generator already offers basic gadgets, which make it possible to
change data in the memory or to call functions. This makes it easy to
create or adapt ROP chains, which are also available in a readable
format. An exemplary creation of a very basic ROP chain can be seen in the following:

```
// Creates a basic ROP Chain which copies data from $payload_srcaddr and executes it.
// The problem is finding the correct address for $payload_srcaddr
function generateropchain_type1(){
    global $payload_srcaddr;
    $payload_size = 0x20000;
    $codegen_addr = 0x01800000;
    // Switch to core 1.
    ropgen_switchto_core1();
    // Copy code to the JIT area
    ropgen_copycodebin_to_codegen($codegen_addr, $payload_srcaddr, $payload_size);
    ropchain_appendu32($codegen_addr); // Execute code in the jit area
}
```

This has to be extended by the part of finding/placing a valid copy of the payload in memory, so it can be copied to the JIT-area and executed.

For this purpose a function has been added which allows to write a payload (which will be embedded in the ROP-Chain) directly into memory. The corresponding function can be found in the listing below.
```
function ropgen_writerop_toAddress($path, $dstaddr){
    $payload = wiiuhaxx_loadfilebinary($path);
    $len = strlen($payload);
    for($i = 0; $i < $len; $i +=4) {
        ropgen_writeword_tomem(hexdec (bin2hex (substr($payload, $i, 4))),$dstaddr + $i);
    }
}
```
This payload length is limited by the maximum length of the ROP-Chain. The used payload is just 88 bytes and it's used to find the "real" payload in memory and copy it to a pre-defined address. 
A existing gadget can be used to place arguments for the payload into the registers 24-31. After the payload is writting into memory, it will be copied to the JIT-Area, the registers will be set and the payload will be execution.
An excerpt of the ROP Chain can be found in the listing below.

```
// Write our small search payload somewhere into mem
ropgen_writerop_toAddress($wiiuhaxxcfg_searchpayloadfilepath,$payload_tmp_address);

// Copy it to codegen/jit area
ropgen_copycodebin_to_codegen($codegen_addr, $payload_tmp_address, $search_payload_length);

// Set up some parameters
$regs = array();
$regs[24 - 24] = $ROP_OSFatal;//r24
$regs[25 - 24] = $ROP_Exit;//r25
$regs[26 - 24] = $payload_size;//r26 sizeToCopy
$regs[27 - 24] = $payload_search_for - 0x04;// r27 SearchFor. substract 0x4 so we didn't find THIS accidentally.
$regs[28 - 24] = $payload_start_search;  //r28 start of search
$regs[29 - 24] = $valid_payload_dst_address ; //r29 target address
$regs[30 - 24] = 0x8;//r30 The payload can do this at entry to determine the start address of the code-loading ROP-chain: r1+= r30. r1+4 after that is where the jump-addr should be loaded from. The above r29 is a ptr to the input data used for payload loading.
$regs[31 - 24] = $ROPHEAP;//r31    
ropgen_pop_r24_to_r31($regs);//Setup r24..r31 at the time of payload entry. Basically a "paramblk" in the form of registers, since this is the only available way to do this with the ROP-gadgets currently used by this codebase.

// And run it!
ropchain_appendu32($codegen_addr); //Jump to the codegen area where the payload was written.
```

The actual implementation can be found [here](https://github.com/wiiu-env/wiiuhaxx_common/blob/master/wiiuhaxx_searcher.s).
Afterwards the "real" payload can be copied from the pre-defined address to the JIT-Area, where it can be executed from. See:

```
// On success, we should now have our actual payload @valid_payload_dst_address. Lets copy it to codegen.
ropgen_copycodebin_to_codegen($codegen_addr, $valid_payload_dst_address, $payload_size);
// and run it!
ropchain_appendu32($codegen_addr);
```

The full ROP-Chain used in this implementation can be found [here](https://github.com/wiiu-env/wiiuhaxx_common/blob/master/wiiu_browserhax_common.php#L533).

In addition, a new plattform independent [rop-gadget-finder](https://github.com/wiiu-env/RPXGadgetFinder) has been implemented in Java, which is used to the address of gadgets in system libraries either by a binany pattern or function name. 
This new finder uses [YAML](https://en.wikipedia.org/wiki/YAML) files to specify which gadgets should be searched. An example config for gadgets of the coreinit.rpl can be found [here](https://github.com/wiiu-env/wiiuhaxx_common/blob/master/coreinit.yml). This 

# Browser exploit payload

The browser exploit can be used to execute any payloads that fit into
the JIT area (32 KiB). Now a new payload should be implemented that has the goal of being able to execute
another, larger payload from the SD card.

To be able to implement this, a kernel exploit is required, which is
explained in the next section. The kernel exploit is required to be able remain code execution while switching to an application which has access to the SD Card.

## Kernel-Exploit

The kernel exploit takes advantage of the fact that the GPU directly
accesses physical memory and also has access to parts of the kernel
area. Applications are normally not allowed access to the kernel area.

The kernel exploit is already publicly available and is compatible with
the current version of the OS (5.5.X). The implementation presented here is a
slight modification of the [existing implementation](https://github.com/wiiudev/libwiiu/blob/master/kernel/gx2sploit/src/loader.c). However, the
basic idea, the used values and the exploited bugs have been adopted.

The basic idea is to manipulate the kernel heap to allocate a memory
block that can be accessed by an application. To achieve this, some
information about the kernel heap is helpful. Information about the
kernel can be obtained from the open source emulator
[decaf](https://github.com/decaf-emu/decaf-emu\), in which the kernel was
reverse-engineered.

-   The heap implementation is a [`tiny heap`](https://github.com/decaf-emu/decaf-emu/blob/002ee0cebfc8f5d724d02cb92e65bb4249e90b0a/src/libdecaf/src/cafe/cafe_tinyheap.cpp).
-   If memory is being allocated, the blocks are iterated until there is
    a free block. ([source](https://github.com/decaf-emu/decaf-emu/blob/002ee0cebfc8f5d724d02cb92e65bb4249e90b0a/src/libdecaf/src/cafe/cafe_tinyheap.cpp#L375-L487))
-   This search starts with a block defined in a variable, which is
    referred to in the following as *firstBlockIdx*.
-   The position of the metadata of this memory block is calculated
    using the variable *firstBlockIdx*.
-   Due to the absence of checks, the manipulation of the variable
    *firstBlockIdx* will allow the next memory block to be allocated in
    a memory area that is accessible to applications.
-   The memory address of the variable *firstBlockIdx* is known and does
    not change.

If it is possible to manipulate the variable *firstBlockIdx*, the meta
information of a memory block allocated can be manipulated. The data
structure of the meta information is called [TrackingBlockBase](https://github.com/decaf-emu/decaf-emu/blob/002ee0cebfc8f5d724d02cb92e65bb4249e90b0a/src/libdecaf/src/cafe/cafe_tinyheap.cpp#L13-L34) in
decaf and is represented in the Listing below. The field data refers to the address of the
block to be allocated. If it points to memory that can be modified by
applications, memory used in the kernel can be manipulated.

```
struct TrackingBlockBase {   
   void * data; // Pointer to the allocated memory of the block   
   int32_t size; // Size of allocated memory, negative if not yet allocated
   int32_t prevBlockIdx; // Index of the previous block  
   int32_t nextBlockIdx; // Index of the following block
};
// TrackingBlockBase struct
```

A manipulation of the variable *firstBlockIdx* can be achieved via the
GPU. The function `SetSemaphore` allows to use a semaphore in a given
address in memory. It is possible to use an address in the kernel heap.
Effectively the variable *firstBlockIdx* can be manipulated by
incrementing the semaphore at the corresponding position in the memory.
The system library of the graphics card offers the possibility to use
the function. To bypass operating system checks, a corresponding PM450
package is generated directly and sent to the graphics card via the
system libraries.

A PM4 packet of the type [`MEM_SEMAPHORE`](http://developer.amd.com/
wordpress/media/2013/10/R6xx_R7xx_3D.pdf) is
created, which receives the physical address of the *firstBlockIdx*
variable as destination address. Each of these packets increases the
value of the variable and moves the position from which the
metainformation (TrackingBlockBase) of the next memory block is read.

```
uint32_t* pm4 = (uint32_t*)MEMAllocFromDefaultHeapEx(0x20, 0x1000);
pm4[0] |= 0xC0013900; //PACKET3_MEM_SEMAPHORE
pm4[1] |= kpaddr;     //ADDR_LO = target
pm4[2] |= 0xC0000000; //SEL semaphore signal
pm4[3] |= 0x80000000; //nop
pm4[4] |= 0x80000000; //nop
pm4[5] |= 0x80000000; //nop
pm4[6] |= 0x80000000; //nop
pm4[7] |= 0x80000000; //nop

DCFlushRange(pm4, 0x20);

GX2DirectCallDisplayList((void*)pm4, 8 * sizeof(uint32_t)); // increment value of kpaddr by 0x01000000
GX2DirectCallDisplayList((void*)pm4, 8 * sizeof(uint32_t)); // increment value of kpaddr by 0x01000000

GX2Flush();
```

After enough packets, the meta information is read from a predictable
address that can be manipulated by applications. In this case, the metadata is read from 0xFF200014 instead of 0x1F200014.

```
/* Allocate memory accessible to applications that is stored in the kernel
   is to be used */
uint32_t *drvhax = OSAllocFromSystem(0x4c, 4);

/* Prepare kernel heap entry */
struct TrackingBlockBase *metadata = (struct TrackingBlockBase*) 0x1F200014;
metadata.data = drvhax;
metadata.size = -0x4c; // negative -> is still available
metadata.prevBlockIdx = (uint32_t) -1;
metadata.nextBlockIdx = (uint32_t) -1;
```

The listing above shows the creation of the wrong
TrackingBlockBase entry at the position expected by the kernel heap from
manipulating the *firstBlockIdx* variable. Memory is allocated in line 3
and assigned to the entry in line 7. The negative size of the size field
in line 8 indicates to the kernel heap that this memory block is not yet
being used. 

This makes it possible to manipulate objects used in the kernel using an
application. The next time something on the kernel heap is allocated, the memory
available to userspace is used, which can be controlled.
In *Cafe OS*, it's possible to register as `OSDriver`. This `OSDriver` will be registered as hooks into system events.
Whenever a Driver gets (de-)initialized or the main application acquires/released the foreground, a function of that Driver will be called.
However, the heap for userspace will be cleared whenever the application switches. 
To allow an `OSDriver` to store persistent data, an mechanism exist, to store this data on the kernel heap.
This is the so-called `SaveArea` of a `OSDriver`. Before an application closes, the Driver can store data in the `SaveArea`, and load it back when a new application loads.

The data of the registered `OSDriver` is allocated on the kernel heap. 
Using the technique described above, it is possible to allocate the 
`OSDriver` data to an area accessible from userspace (`0x1F200014`) and then manipulate it.

The position of the `SaveArea` can be freely selected by manipulating the
`OSDriver` data. The data is written with kernel privileges, which makes
it possible to set the `SaveArea` to a position within the kernel. Afterwards
data can be written to the position via the function `CopyTo_SaveArea`.
Listing below shows how any memory area can be modified
using a registered driver.

```
/* Register driver. The driver data is stored by the
   manipulated kernel heap into the previously allocated memory.
   `drvhax`. */
char drvname[6] = {'D', 'R', 'V', 'H', 'A', 'X'};
Register(drvname, 6, NULL, NULL);

/* Via the driver data, the address of the _SaveArea_
   can be manipulated. */
drvhax[0x44/4] = ANY_ADDRESS;
/* SIZE bytes are copied from DATA to ANY_ADDRESS. */
CopyTo_SaveArea_(drvname, 6, DATA, SIZE);
```

From here, Syscalls can be registered for reading and writing with
kernel privileges. For this, parts of the existing kernel code are used.
Any writing and reading of data with kernel privileges is now possible.

The full used kernel exploit implementation can be found [here](https://github.com/wiiu-env/gx2sploit).

### Payload implementation

The payload, which is executed via the browser exploit, can be divided
into two parts. On the one hand the exploit specific part, which differs
depending on the entry point, on the other hand, common exploit
independent tasks can be abstracted into another payload. This applies
in particular to the part responsible for loading and copying
`payload.elf` from the SD card.

The initial payload for the browser is to run as follows and performs
the following actions:

-   A new thread is created and used.
-   The processes of the browser are terminated so that the application
    can be switched in the future.
-   By executing the kernel exploit, syscalls can be registered for
    reading and writing with kernel privileges. These syscalls use
    existing code in memory and are therefore permanently available as
    syscalls `kern_read` (0x34) and `kern_write` (0x35).
-   The `kern_write` syscall can be used to register a custom Syscall
    `KernelCopyData` that copies data independently of the MMU. Existing
    access restrictions can thus be circumvented. The implementation can
    be taken over from existing solutions.

A list of already used syscalls can either be read from the kernel or
taken from the [WiiUbrew wiki](https://wiiubrew.org/wiki/Cafe_OS_Syscalls).

At this point, all requirements are met to switch to another application
while code execution remains intact. For this, the second,
exploit-independent, payload is copied with the help of the
`KernelCopyData` syscall to a free space in the memory that is executable.
Theoretically, you could already set and use your own memory areas at
this point, but as few changes as possible are deliberately made. The
goal is to make it possible to load a payload from the SD card with as
few changes to the system as possible, so that the payload can expect a
standardized, already known environment.

A generic payload is then copied via the `KernelCopyData` syscall to the
location in memory where it is statically linked. It is known from
existing solutions that the memory area between 0x011DD000 and
0x011E0000 is not used and can be executed as well. In order for code to
be executed after the application change, the system is modified so that
the call to the main function is overwritten by a jump to the payload.
In the following, this payload is called `main_hook.elf`. The address of
the actual main function can later be read from memory and the original
function, if necessary, executed.

At this point, a change to any application would execute the payload.
This is the case until the main hook is reverted.

Hthe access to the JIT area, in which the code of the `KernelCopyData` 
syscall is located, is not longer given. This means that
it can no longer be used. At the same time there is the problem that the
free memory area from 0x011DD000 to 0x011E0000, in which the
`main_hook.elf` is located, code can be executed, but not with kernel
privileges. Thus it is not possible to use a syscall whose code is in
this range.

To work around this, the necessary permissions must be set and execution
with kernel privileges in this area must be allowed. These are
implemented by the MMU and managed for example by BAT
registers (Block Address Translation Registers). They can be used to map physical memory block by block to
virtual memory with corresponding permissions. There are two types of
BAT registers: Data BAT registers (DBAT), which map the memory in terms
of data, and Instruction BAT registers (IBAT), which map the memory in
terms of execution.

In order for the used memory area to be executed with kernel privileges
and thus registered as a function as syscall, a corresponding IBAT
register must be set. Therefore a function is placed in a free region of
the PowerPC kernel (at `0xFFF02344`) and registered as a Syscall (0x09). 
With this function the IBAT0 can be set. In this case a function is used to overwrite the
zeroth IBAT entry. Theoretically any other IBAT entry can be used. A
syscall to read out the previous value is possible, but not mandatory.
The `main_hook` can use the syscall 0x09 with the following function signature:

```
extern void SC_0x09_SETIBAT0(uint32_t upper, uint32_t lower);
```

Now all preparations have been made to switch to the *Mii Maker*
application. At the start of the application the `main_hook.elf` payload
is executed.

The following guarantees can be given to the payload:

-   The payload is called every time an application is started.
-   The payload has access to the SD card.
-   A syscall 0x09 for setting IBAT0 is available.
-   The `kern_read` (0x34) and `kern_write` (0x34) syscalls can be used to read or write
    with kernel privileges.

A similar procedure will be implemented for all further entry points in
the future. If the same guarantees are given, the `main_hook.elf` payload
can be used without any modifications. An example implementation for Browser
Exploit payload can be found [here](https://github.com/wiiu-env/JsTypeHax_payload).

### Loading a gerneric payload (`main_hook.elf`)

In the `main_hook.elf` the loading of another payload, the `payload.elf`,
from the SD card shall be implemented. This requires that the
`main_hook.elf` payload has access to the SD card, the syscalls for
reading and writing with kernel privileges, and the syscall for setting
the IBAT register. These requirements are fulfilled using the presented browser exploit payload.

To be able to register the functions of the `main_hook.elf` as syscall,
the IBAT0 register must be set via the corresponding syscall. The
original mapping of the used memory area is retained and only execution
rights with kernel privileges are added. Then a syscall can be
registered, which defines an own, 8 MiB memory area, with full read,
write and execution permissions. The syscall is executed on every core
of the CPU where it should be available.

```
// Call this on each CPU Core
static void SCSetupIBAT4DBAT5() {
    asm volatile("eieio; isync");

    // Give our and the kernel full execution rights.
    // 00800000-01000000 => 30800000-31000000 (read/write, user/supervisor)
    unsigned int ibat4u = 0x008000FF;
    unsigned int ibat4l = 0x30800012;
    asm volatile("mtspr 560, %0" : : "r" (ibat4u));
    asm volatile("mtspr 561, %0" : : "r" (ibat4l));

    // Give our and the kernel full data access rights.
    // 00800000-01000000 => 30800000-31000000 (read/write, user/supervisor)
    unsigned int dbat5u = ibat4u;
    unsigned int dbat5l = ibat4l;
    asm volatile("mtspr 570, %0" : : "r" (dbat5u));
    asm volatile("mtspr 571, %0" : : "r" (dbat5l));

    asm volatile("eieio; isync");
}
```

The `payload.elf` is then loaded into this newly defined memory area. The
loading of the ELF file can be done by using existing code from the `Homebrew Launcher`.
This allows you to load and execute ELF files that are located on the SD
card and have a maximum size of 8MiB. If the SD card cannot be accessed
or there is no `payload.elf` on the SD card, all previous changes are
reversed and the system menu is loaded.

Theoretically, the definition of the memory area could be overwritten by
the operating system at any time. In order to keep as little logic as
possible in the `main_hook.elf`, the responsibility to prevent this from
happening lies with the loaded `payload.elf`.

An example implementation of the `main_hook.elf` can be found [here](https://github.com/wiiu-env/payload_loader).

# Backwards compatiblity
The`payload.elf` is an abtract payload and can be used for anything. In order to have backwards compatiblity to the old existing homebrew environment, 
I ported the homebrew launcher installer as a `payload.elf`, which can be found [here](https://github.com/wiiu-env/homebrew_launcher_installer).
With this it's possible to have the stable browser exploit with abstract payload loading, but still using the old environment until the new one is finished.