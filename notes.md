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
TODO reduce to the size of an abstract, add usage models as an introduction should do, what are the current solving solutions

Mobile devices and terminals have for a long time followed this usage model: one device for one user with one specific usage. The advent of modern mobile operating systems allowed one same device to answer many usages, based on the amount of available software: productivity, media player, camera, games, etc... This has broaden considerably the usage model of mobile devices, which are able to complete multiple tasks under multiple environments. On another hand, tablet computers and increasingly capable smartphones have rapidly replaced personal computers for many specific usages like browsing web, watching movies, sharing activities or play games together: such mobile devices are more and more likely to be used by multiple people. Therefore, this trend has started to shift to "one device for multiple users with many usages". However, mobile device and their operating systems only focus on the single user experience; the leading OS such as Android and iOS are no exception, and such have not yet transitioned.

The problem of many people using one platform has been answered long ago on traditional desktop computers, as they experienced the same transition: commodity OS like Windows, Mac OSX or any Linux-based desktop distribution have introduced the multi-session feature, and ensure that multiple people can use one computer, with each user having its own privacy, data and environment. On another hand, one user which want to use multiple different environments has the choice of running multiple OS on their personal computer, through simple solutions such as multi-boot or virtual machines. Such solutions of course are inexistant on mobile devices, either because of inexisting OS support, or the lack of hardware support; at most, a user could multiple devices instead, or risk to expose private data if other people uses its device. This lack of support is understandable, as the shift in the usage model of mobile devices has emerged only recently.

We propose Fastswitch, a solution to allow any number of Linux-based OS to time-share a mobile device: the user can use multiple environments with multiple configurations, at the same time. The user can switch very quickly from one OS to another. Each OS is independent and cannot access other running OS data (_actually, if root, they can_). Fastswitch is not a virtualization-based method, does not need any special hardware support, or any bootloader modifications, and can be readily implemented in any Linux-based smartphone or tablet computer. Fastswitch has been implemented on a Galaxy Nexus ARM-based smart-phone running Android 4.0 (Linux 3.0.8) and Android 4.1 (Linux 3.0.31).

Section 2 describes the mechanism behind Fastswitch. Section 3 discusses the implementation and evaluation results. Section 4 describes related works, and we conclude this paper in the section 5.



### Section 2 Mechanism

The term _platform_ will refer to a mobile device or terminal such as a tablet computer, a smart-phone, or any embedded system. The term _device_ will refer to a hardware component of a platform, such as screen, radio, wifi, etc.

#### Execution unit: Instance
Fastswitch introduces a framework which allows multiple Linux-based OS, referred as _instance_, to time-share one platform: at any time, one instance is _running_ while all other instances are _sleeping_. Three rules describes Fastswitch:

1.  The _running_ instance has full control over all hardware devices including screen and any input device. Therefore, at any time, it is the only OS the user sees, and operates.  
    This condition reflects the expectation to have the full power and capabilities of the platform when using it. 

2.  The _running_ instance can go to _sleeping_ state, and start a new instance, or call any _sleeping_ instance to become the new _running_ instance in a procedure we call _switching_. The _sleeping_ instances are loaded in memory, must be ready to be called anytime, but can not wake up by themselves.  
    This is our current answer to enable one device to have multiple usage. Each instance can be tailored to suit specific needs, and be called forth when necessary.  

3.  Any instance must not interfere with any other instance. (_work in progress??_) The primary concern behind this is privacy.  
    Any instance can not natively access the data of other loaded instances, although malicious software can explicitly inspect physical memory or mount filesystems. (_to put in drawbacks of solution?_)  
    A second concern is boxing: A crashing kernel may actually crash the whole device, resulting in losing all instance data. (_to put in drawbacks of solution?_)

While hardware devices are under the control of the _running_ instance, RAM and non-volatile storage are shared between each loaded (_running_ or _sleeping_) instance. This is a consequence of the third rule. We make sure that each instance only have access to the memory it owns (more details in sub-section _Memory sharing_).


#### Sleeping state
We are using the set of suspend and resume operations supported by Linux (referred as _suspend-to-RAM_ in Linux terminology) to put an instance into a _sleeping_ state. It ideally corresponds to a very low power state where only RAM remains powered on. Platforms with aggressive power management policy will, during suspend operations, save most if not all devices state and CPU context in RAM and power them off (or put them in a low power state), while the resume operations will power them on and restore their state. 
The platform, on an external event triggered by user interaction or a hardware device, can resume from a suspended state by retrieving necessary data from the RAM, and re-initializing CPU and device state afterwards. Suspend and resume operations are a very common power management feature found in mobile devices, where energy saving is a critical factor; most hardware platforms are therefore ready to support Fastswitch.


#### Switching
The core function of Fastswitch is the ability to switch from one instance to another. In essence, the idea is simple: we use the suspend operations on the _running_ instance, then use the resume operations on any _sleeping_ instance we want to bring forward. However, resuming an instance can be done only under two conditions:

- During a switch, the control over hardware devices is expected to be handed from the _running_ instance to the _sleeping_ instance we want to switch to. This means the _running_ instance must set the devices in a state where the _sleeping_ instance can recover from, that is, the state the devices were left when the _sleeping_ instance previously suspended. Instances do not necessarily stem from the same Linux-based OS, nor must they be using the same kernel; as long as involved instances set devices to a compatible state (the simplest state being powered off), switching is possible. _TODO to speed things up, we may not need to suspend everything, actually_

- A combination of (memory mapped) registers must be set with the correct values, i.e. the values that were written when the instance went to sleep. Those registers are platform-dependant, and may include for example the physical address to assign to the program counter once the platform wakes up from suspend, or the physical addresses of MMU tables (TODO add more relevant examples). Each instance is very likely to have different values for some of those registers; If we were to resume an instance with incoherent values, the resume operation would fail, and result in exceptions, crash, memory corruption, or unpredictable effects. We choose to call those registers _mandatory resume registers_ in the next part.

As an implementation detail, an instance at boot will reserve a chunk of memory for its own usage: this chunk, referred next as _instance page_, is unique to this instance and is valid as long as the instance is loaded in memory. The _instance page_ will hold the necessary data and parameters regarding this instance.


Switching instance involves the following steps:
1.  On boot, we register two critical pieces of information into the _instance page_: the physical addresses of _mandatory resume registers_, and the physical address of the first resume routine (i.e. the first kernel code instruction that the CPU executes once the platform wakes up).
2.  A _running_ instance initiates a switch request, and indicates which _sleeping_ instance the user want to switch to. At the next opportunity to suspend, the OS will execute suspend operations, and enter a low power state.
3.  An event will wake up the platform. At this stage, ROM code internal to the platform may perform initialization routines to properly wake up the platform. As soon as the platform exit the low power state and jumps to the _running_ instance _resume routine_:
    -   We save the content of the _mandatory resume registers_ into the _running instance page_
    -   We load the content of the _mandatory resume registers_ from the _sleeping instance page_
    -   We jump to the _sleeping_ instance resume routine
4.  At this point, we have prepared the platform so the _sleeping_ instance can resume properly. The resume operations of the _sleeping_ instance can proceed.

This process results in the _running_ instance becoming a _sleeping_ instance, and the target _sleeping_ instance becoming the _running_ instance.


#### Booting a new instance
Fastswitch allows the user to start new instances at runtime. Booting a new instance requires the hardware devices to be in a proper state, where a fresh OS can boot from.  
We could setup the hardware state ourselves to accommodate a booting OS: This requires accurate knowledge of how the devices are initialized by the bootloader. This information, however, may not be open, especially in proprietary environment, and thus is not an acceptable solution. This is the case in the platform we used in our prototype.  
Therefore, we have chosen to let the bootloader initialize properly the devices. This method, without any knowledge of what the bootloader does, ensure that the new instance will boot as if the platform and the bootloader launched it.

We assume for now that the instance we want to load and the _running_ instance have distinct memory ranges, so they don't overlap. This problem is discussed and solved in the sub-section _Memory sharing_.

The _initial_ instance refers to the OS loaded by the bootloader on normal operation.


The process involves several steps:
1.  The _running_ instance initiates a boot request: Fastswitch loads into an arbitrary memory location the necessary files and data to boot the target instance, just like a bootloader would have done. The target location must be unused by the _running_ instance.
2.  At the next opportunity to suspend, the OS will execute suspend operations. The _running_ instance becomes a _sleeping_ instance. Once the devices have their state and context saved, we initiate a warm-reset order directly to the CPU. This is the way we employ to jump back to the bootloader. We assume that the bootloader will not erase or clean the whole RAM.
3.  Once reset, the platform will execute ROM code and bootloader code. Both code will properly initialize the devices so a fresh OS can boot. The bootloader will also load several files into memory, in particular the kernel compressed image of the _initial_ instance, then jump to its address.
    *   Since the bootloader overwrites several memory ranges by loading files and data, we have to make sure to forbid any instance to use those location.
4.  On normal operation, the decompressor code placed at the head of the compressed image would have decompressed it. But when we initiate a boot request, we modified the same decompressor code to jump to the target instance image instead.
5.  We proceed by executing the code of the target instance image. The target instance will then boot normally.

This process results in the former _running_ instance becoming a _sleeping_ instance, ready to be called back. At its place, the new instance becomes the _running_ instance.


### Section 3 Memory sharing
To allow flexibility in the amount of memory available for each instance, we propose to use the _Memory hotplug_ feature of Linux.

This feature has been designed with two goals in mind:
1.  Changing the amount of available memory in a system.
2.  Allow the partial replacement of failing memory hardware without shutting down the machine. This is primarily targeted at data centers and servers, where downtime may have severe implications and should be avoided if possible.

We leverage its capability to free memory for a new instance, or exchange memory between running instances.


#### Principle
The principle behind _Memory Hotplug_ relies on two underlying features of Linux.

The most common memory model, referred as _Flat memory model_ in the Linux terminology, is to consider the physical RAM as a single physical contiguous memory area, with one starting address and one ending address. Multiple banks of RAM therefore have to be placed at successive addresses (which is the case in non-NUMA machines), and constitute together one single RAM area. The _Sparse memory model_, by contrast, takes a different approach and divides physical memory space equally into memory sections of a fixed size. The size of one section may depend on the architecture, but can be user defined. __TODO may add a picture to explain this__

The physical RAM therefore does not need to be a single contiguous physical address range; even if the sections are not adjacent, the OS will still map them so we have a contiguous virtual memory space. The _Sparse memory model_ allows the OS to see memory not as a monolithic unit, but as a set of multiple units, characteristic necessary for the _Memory hotplug _feature. __TODO may need to find a performance report of sparsemem__

_Memory hotplug_ introduces a logical state of a memory section as seen from the OS point of view: a section is _online_ or _offline_. An online section is mapped by the OS and the physical pages of the section are counted in the pool of available pages, while an offline section is mapped by the OS but its physical pages are not counted in the pool of available pages, therefore preventing the OS from using physical pages belonging to this section. The logical state of a section is unrelated to the physical presence of the RAM bank in the system (which in our case is always present), and is purely used to allow or disallow an OS to use the range of physical memory associated to this section.

What makes the feature interesting in our case is its capability to change the state of a section at runtime: from offline to online (hot-add), or from online to offline (hot-removal).

When a section state is changed to online, all the pages belonging to the section are added to the pool of available pages, and are marked as free. When a section state is changed to offline, the _Memory hotplug_ feature relies on the _Memory migration_ mechanism to copy the pages of the section to another physical location, and changes the references to the moved page to the new location. __TODO to study exactly.__ Of course, not all pages can be migrated. If such a page happens to be in a section we want to set offline, the operation will fail. To prevent this, _Memory hotplug_ allows us to mark a memory zone as movable: if a section belongs to a movable zone, then the OS will assume that this section can be set offline anytime, and so will avoid putting unmovable pages into it. Otherwise, it will not belong to the movable zone, and the OS does not guarantee that it can be set offline. It is for example the case for sections which contains kernel code (which can not be moved elsewhere, obviously). The current implementation of _Memory hotplug_ allow us to define how much memory belongs to the movable zone. 


#### Sharing
In the following part, we assume that the physical RAM is a single contiguous memory area.  
The term _memory block_ refers to a single contiguous memory area, included in the physical RAM, without any constraint in the starting and ending address (except for page alignment).  
The term _memory section_ refers a section as defined in the _Sparse memory model_, that is, a single contiguous memory area, included in the physical RAM, but with constraints in the size, starting and ending address.

An OS expects to see on boot all the physical RAM it _can_ use, and will do several platform-specific operations:
    - Remove memory blocks from the System RAM: the kernel will never read or write those blocks, so specific drivers can use them freely
    - Reserve portions of the memory
    - Register and map every memory block the OS finds

In Fastswitch, we do not change this behavior: we must expose every memory blocks an instance expect to see. All instances on boot will perform the previous actions, so they can _eventually_ use those blocks. This means that each instance could freely change the state of any section. Two cases are possible:
1.  One section is mandatory for each instance. For example, one device driver needs to access a specific memory chunk reserved for it: a common example may be the screen framebuffer. In that case, such section must be shared: it is online for each instance. It also must not be set offline (_this operation would not make much sense, but if it succeed, the pages contained in this section would migrate elsewhere_), and any attempt to set it offline must fail.
2.  In most cases, one section must not be used by two instances at the same time. It is at most online for one instance, and offline for the others. Otherwise, memory corruption would ensue.  

To enforce those rules, we introduce the concept of "section ownership" to ensure that each section belongs to at most one instance, or is shared among all. We use a table common to all instances to keep track of the "owner" of each section. The ownership is different from the state of a section (online or offline). For example, a section owned by an instance can be online or offline (and it is neceassrily offline for any other instance). An instance can not change the state of a section if it is not the section's owner. This protection ensure that each section appears online to only one instance (its owner) at most, excluding the shared sections.


#### Booting an instance
We explained in the previous section, what happens when the user requests to boot a new instance. The following part provides additional details regarding how memory is allocated to the new instance.  

It is very likely that the new instance has memory layout requirements, such as a contiguous physical memory space of a certain size. The _running_ instance therefore needs to have at least enough sections in its movable zone to allocate to the new instance, and those sections must conform to the expected memory layout.

1.  The _running_ instance must set offline any section needed by the new instance, and change the ownership of those sections to the target instance.
2.  It must then copy in those newly freed sections the necessary files and data, such as kernel image.
3.  The procedure continues, as described in Section 2, until the new instance is ready to boot.
4.  During the memory initialization (described in sub section _Sharing_), the new instance detects and maps every memory section. Then it checks which sections belong to it: all other not-shared sections are set offline, so the new instance will not accidentally overwrite them.
5.  The new instance completes the boot procedure.

On Linux-based OS, the whole procedure is guaranteed to not touch to sections owned by other, because any code and allocated data structure are all within the kernel space (__TODO looks for a better formulation, it is actually bootmem__) and not in sections belonging to others.


#### Section transfer
_Memory hotplug_ allow us to dynamically tailor the amount of memory available to each instance. Consider the following case: the _running_ instance has not enough memory to run a specific application. To solve this memory shortage would require to set online a few offline sections. If no offline sections are available (e.g. all offline sections belongs to other _sleeping_ instances), Fastswitch allows an instance to request sections to other instances. 

The procedure goes like this: the _running instance_ issues a memory section request, indicating the number of sections it would like. At the next opportunity to suspend, the _running_ instance will execute suspend operations, and the first _sleeping_ instance will resume. Upon resuming, it will try to set offline any available sections (i.e. belonging to its movable zone) then suspend itself. If the number of successfully freed sections does not reach the requested number, another _sleeping_ instance is called, and will attempt to free more sections. This procedure continues until he requested number of sections have been freed, or all _sleeping_ instances have tried to free section. Regardless of the outcome, the requesting instance resumes at last, and claims the ownership of any freed section.




Trash
=============

Since there is only one _running_ instance which make memory operations, we are not concerned by the coherency of the cache and memory..



    Performance and limitations to put in performance section

Since each instance runs directly on hardware, no performance penalty is imposed on the OS, and device drivers don't need to be modified. However, just like virtualization, RAM is the limiting resource as any loaded instance is occupying a significant part of the RAM.



To cater the need of multiple usage, users may have multiple devices (for instance, holding a personal smartphone and a business smartphone). To cater the need of ultiple users on Android-based platforms, there have been examples of software solutions introducing a "multi-session" look-alike feature, each "session" possessing their own environment.

 Some memory ranges are expected to be shared between all instances only under a few conditions (usually related to device drivers).