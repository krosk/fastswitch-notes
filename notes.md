# Fastswitch organized notes
Roughly edited document to organize the concept and ideas behind Fastswitch  
Author: Alexis Hé

***

## Fastswitch: time-sharing one device between multiple Linux-based OSes
(tentative title)

### Rough plan

Abstract
> Talk about how multiuser and multiusage can raise how we can use mobile devices, talk about the solution we present, Fastswitch, directly usable on Android OS without any hardware virtualization support

Introduction
> Talk again about how multiuser and multiusage, new usage models, are becoming a factor.  
> Talk about existing solutions :
- multi boot, its drawbacks
- virtualization, lack of support, extensive work needed
> Our solution, rely on support of OS.

Fastswitch mechanism
> Describe the overall structure of how it works
> Basic mechanism to go from one instance to another instance, using the suspend operation
> Memory framework

Implementation
> Done on Galaxy Nexus, booting two different kernels (3.0.8 ICS, 3.0.31 JB) and two different instances.
> Switching time, 1.2 sec, heavily reliant on device
> No overhead, or at least memory overhead

Limitations
> rely on robust suspend operation
> RAM is the limiting factor

Future works
> Multi core, how about doing small tasks by a second operating system in parallel

Related works
> BOMLO, how we don't rely on bootloader, leverage the memory access problem,
> Virtualization-like support (cells, micro kernel, etc...)
> Cells: we are able to run more systems, different versions of android, different kernels,

Miscellaneous
> What about privacy?



### Section 1 Introduction
TODO reduce to the size of an abstract, add usage models as an introducion should do

Mobile devices and terminals have for a long time followed this usage model: one device for one user with one specific usage. The advent of modern mobile operating systems allowed multiple usages on the same device based on the amount of available software: productivity, media player, camera, games, etc... On another hand, tablet computers and increasingly capable smart-phones have rapidly replaced personal computers for many specific usages like browsing web, watching movies, sharing activities or play games together. Therefore, this trend has started to shift to "one device for multiple users with many usages". However, mobile device operating systems only focus on the single user experience, and the leading OS such as Android and iOS are no exception.

We propose Fastswitch, a solution to allows any number of Linux-based OS to time-share a mobile device: the user can use multiple environments with multiple configurations, at the same time. The user can also switch very quickly from one OS to another. Each OS is independent and cannot access other running OS data (_actually, if root, they can_). Fastswitch is not a virtualization-based method, does not need any special hardware support, or any bootloader modifications, and can be readily implemented in any Linux-based smartphone or tablet computer. Fastswitch has been implemented on a Galaxy Nexus ARM-based smart-phone running Android 4.0 (Linux 3.0.8) and Android 4.1 (Linux 3.0.31).

Section 2 describes the mechanism behind Fastswitch. Section 3 describes how physical memory is shared. Section 4 discusses the implementation and evaluation results. Section 5 describes related works, and we conclude this paper in the section 6.



### Section 2 Mechanism

_The term "platform" will refer to a mobile device or terminal such as a tablet computer, a smart-phone, or any embedded system. The term "device" will refer to a hardware component of a platform, such as screen, radio, wifi, etc._

Fastswitch introduces a framework which allows multiple Linux-based OS, referred as "instance", to time-share one platform: at any time, one instance is "running" while all other instances are "sleeping". Three rules describes Fastswitch:

1.  The "running" instance has full control over all hardware devices, and is the one we see on screen and are manipulating. 
    This condition reflects the ideal usage of a platform: a user expects to have the full power and capabilities of the platform when using it.

2.  The "running" instance can go to "sleeping" state, and start a new instance, or call any "sleeping" instance to become the new "running" instance. The "sleeping" instances are loaded in memory, must be ready to be called anytime, but can not wake up by themselves. 
    This is our answer to cater the need of multi-usage or multi-user capability. Each instance can be tailored to suit a specific need (low consumption mode, games environment, guest session...).

3.  Any instance must not interfere with any other instance. (_work in progress??_) The primary concern behind this is privacy. 
    Any instance can not natively access the data of other loaded instances, although malicious software can explicitly inspect physical memory or mount filesystems. 
    A second concern is boxing: A crashing kernel may actually crash the whole device, resulting in losing all instance data.



#### Instance boundaries
TODO to add details about why memory may need to be shared. 
While hardware devices are under the control of the "running" instance, RAM and non-volatile storage are shared between each loaded (running or sleeping) instance. This is a consequence of the third rule. We make sure that each instance only have access to the memory it owns (more details in sub-section _Memory_). Some memory ranges are expected to be shared between all instances only under a few conditions (usually related to device drivers).


#### Sleeping state
We are using the set of suspend and resume operations supported by Linux (referred as suspend-to-RAM in Linux terminology) to put an instance in the said "sleeping" state. It ideally corresponds to a very low power state where most if not all devices are powered off, CPU included. Only the RAM remains powered on, so the OS state, devices state and CPUs state are conserved in RAM. The platform, on a signal (IRQ usually), can resume from a suspended state by retrieving necessary data from the RAM, and re-initializing CPU and device state afterwards. Suspend and resume operations are a very common power management feature found in mobile devices, where energy saving is a critical factor; most hardware platforms are therefore ready to support Fastswitch.


#### Switching instance
The core function of Fastswitch is the ability to switch from one instance to another. In essence, the idea is simple: we use the suspend operations on the "running" instance, then use the resume operations on any "sleeping" instance we want to bring forward. However, resuming an instance can be done only under two conditions:

- The "running" instance doing suspend operation must set devices be in a state where the "sleeping" instance doing resume operations can recover from (i.e. the state the devices were, just after the "sleeping" instance previously underwent suspend operations). In the case of mobile devices with aggressive power management policy, the suspend operations will usually save the device state in RAM and power it off, and the resume operations will power the device on and restore the state saved in RAM beforehand make sure it is the case. Each instance does not necessarily stem from the same Linux-based OS, nor must it be using the same kernel. But as long as all instances involved have the same suspend and resume operations, switching instance is safe. If it is not the case, devices drivers may need to be modified in order to properly power off the devices, if possible. may need another formulation

- A combination of (memory mapped) registers must be set with the correct values, i.e. the values that were written when the instance went to sleep. Those registers may include for example the physical address to assign to the program counter once the platform wakes up from suspend, or the physical addresses of MMU tables. Each instance is very likely to have different values for some of those registers; If we were to resume an instance with incoherent values, the resume operation would fail, and result in exceptions, crash, memory corruption, or unpredictable effects. This set of registers is referred as "mandatory resume registers".

Switching instance involves the following steps:
-   An instance, at boot, will reserve in an arbitrary memory address a chunk of memory for its own usage. This chunk, referred next as "instance page", is unique to this instance. On boot, we must also register two critical pieces of information into the instance page: the locations of mandatory resume registers, and the physical address of the resume routine (i.e. the first kernel code instruction that the CPU executes once the platform wakes up).
-   A "running" instance initiates a switch request, and indicates which "sleeping" instance the user wants to switch to. At the next opportunity to suspend (see sub-section Wakelocks), the OS will execute suspend operations, and enter a low power state.
-   An event will wake up the platform. At this stage, ROM code internal to the platform may perform initialization routines to properly wake up the platform. As soon as the platform exits the low power state and jumps to the "running" instance resume routine:
    -   We save the content of the "mandatory resume registers" into the "running" instance page
    -   We load the content of the "mandatory resume registers" from the "sleeping" instance page
    -   We jump to the "sleeping" instance resume routine.
-   At this point, we have prepared the platform so the "sleeping" instance can resume properly. The resume routine of the "sleeping" instance gets executed, completing the resume operations.

This process results in the "running" instance becoming a "sleeping" instance, and the target "sleeping" instance becoming the "running" instance.


#### Booting a new instance
Fastswitch allows the user to start new instances at runtime. Booting a new instance requires the hardware devices to be in a proper state, allowing a fresh OS to boot from. We could setup the hardware state ourselves to accommodate a booting OS: This requires accurate knowledge of how the devices are initialized by the bootloader. This information, however, may not be open, especially in proprietary environments, and thus is not an acceptable solution. This is the case in the platform we used for implementation. Therefore, we have chosen to let the bootloader initialize properly the devices. This method ensure that the instance will boot as if the platform and the bootloader launched it, without any knowledge of what the bootloader does.

We assume for now that the instance we want to load and the "running" instance have distinct memory ranges, so they don't overlap. This problem is discussed and solved in section 3, Physical Memory Sharing.

The "initial" instance refers to the OS loaded by the bootloader.

The process involves several steps:
-   The "running" instance initiates a boot request: Fastswitch loads into an arbitrary memory location the necessary files and data to boot the target instance, just like a bootloader would have done. The target location must be unused by the "running" instance.
-   At the next opportunity to suspend (see sub-section Wakelocks), the OS will execute suspend operations. Once the devices have their state and context saved, we initiate a warm-reset order directly to the CPU. This is the way we employ to jump back to the bootloader.
-   Once reset, the platform will execute ROM code and bootloader code. The ROM code and the bootloader will properly initialize the devices so a fresh OS can boot. The bootloader will also load several files into memory, in particular the kernel compressed image of the "initial" instance, then jump to its address.
    -   Since the bootloader overwrites several memory ranges in memory by loading files and data, we have to make sure to forbid any instance to use those locations beforehand.
-   On normal operation, the decompressor code placed at the head of the compressed image would have decompressed it. But when we initiate a boot request, the same decompressor code will jump to the target instance image instead.
-   We proceed by executing the code of the target instance image. The target instance will then boot normally.



### Section 3 Physical memory sharing
To allow flexibility in the amount of memory available for each instance, we propose to use the Memory hotplug feature of Linux.

This feature has been designed with two goals in mind:
1.  Changing the amount of available memory in a system.
2.  Allow the partial replacement of failing memory hardware without shutting down the machine. This is primarily targeted at data centers and servers, where downtime may have severe implications and should be avoided if possible.

We leverage its capability to free memory for a new instance, or exchange memory between running instances.

#### Principle

The principle behind Memory Hotplug relies on two underlying features of Linux.

The most common memory model, referred as Flat Memory Model in the Linux terminology, is to consider the physical RAM as a single physical contiguous memory area, with one starting address and one ending address. Multiple banks of RAM therefore have to be placed at successive addresses (which is the case in non-NUMA machines), and constitute together one single RAM area.

The Sparse Memory Model, by contrast, takes a different approach and divides physical memory space equally into memory sections of a fixed size. The size of one section may depend on the architecture, but can be user defined.

TODO may add a picture to explain this

The physical RAM therefore does not need to be a single contiguous physical address range; the OS will still map any memory section found so it appears contiguous in the virtual memory space, even if said sections are not adjacent. The Sparse Memory Model allows the OS to see memory not as a monolithic unit, but as a set of multiple units, characteristic necessary for the Memory Hotplug feature.

TODO may need to find a performance report of sparsemem

Memory Hotplug introduces the logical state of a memory section as seen from the OS point of view: a section is online or offline. An online section is mapped by the OS and the physical pages of the section are counted in the pool of available pages, while an offline section is mapped by the OS but its physical pages are not counted in the pool of available pages, therefore preventing the OS from using physical pages belonging to this section. The logical state of a section is unrelated to the physical presence of the RAM bank in the system (which in our case is always present), and is purely used to allow or disallow an OS to use a range of physical memory associated to this section.

What makes the feature interesting in our case is its capability to change the state of a section at runtime: from offline to online (hot-add), or from online to offline (hot-removal).

When a section state is changed to online, all the pages belonging to the section are added to the pool of available pages, and are marked as free. When a section state is changed to offline, the Memory Hotplug feature relies on the Memory Migration mechanism to copy the pages of the section to another physical location, and changes the references to the moved page to the new location. to study exactly.

Of course, not all pages can be migrated. If such a page happens to be in a section we want to set offline, the operation will fail. To prevent this, Memory Hotplug allows us to mark a memory zone as movable: if a section belongs to a movable zone, then the OS will assume that this section can be set offline anytime, and so will avoid putting unmovable pages into it. The current implementation of Memory Hotplug allow us to define how much memory belongs to the movable zone, and is preferably lower than the minimal memory requirements of the OS.

#### Usage

Memory Hotplug allow us to dynamically tailor the amount of memory available to each instance.

In the following part, we assume that the physical RAM is a single contiguous memory area.

The term memory block refers to a single contiguous memory area, included in the physical RAM, without any constraint in the starting and ending address (except for page alignment).

The term memory section refers to a single contiguous memory area, included in the physical RAM, but with constraints in the size, starting and ending address.

An OS expects to see on boot all the physical RAM it *can* use, and will do several platform-specific operations:

    Remove memory blocks from the System RAM: the kernel will never read or write those blocks, so specific drivers can use them freely.
    Reserve portions of the memory
    Register and map every memory block it finds

In Fastswitch, we do not change this behavior: all instances actually see every memory sections. But instead of claiming every memory block, we

    Performance and limitations to put in performance section

Since each instance runs directly on hardware, no performance penalty is imposed on the OS, and device drivers don't need to be modified. However, just like virtualization, RAM is the limiting resource as any loaded instance is occupying a significant part of the RAM.