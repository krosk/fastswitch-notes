### 01/12
**running under realview**
* Solving first the problem of ethernet.
```
smc91c111_write: Bad reg 0:6
```
According to the documentation, this register is readonly, and so there is effectuvely something wrong here. After skipping the register, there is clearly something wrong as it tries to read register 7. That would mean a incomplete implementation of the nic (the case 0:6 does not exist), and therefore I should maybe disable it in u-boot?

I suspect the issue is within u-boot. I disabled the nic with -net none, and it seems to work more or less as I at least have access to the command line in u-boot.

Next, in qemu console, use 
```
info mtree
```
to have an overview of the memory map. From my observation, I only see 0x10000000 memory, which is 256Mb of ram. Don't go higher than this, because it overlaps with other things. EB does not support more memory anyway, so this will get dumped.

Next, u-boot is configured with initial environments variables:
```
bootargs=root=/dev/nfs ip=dhcp mem=128M console=ttyAMA0 video=vc:1-2clcdfb: nfsroot=10.1.77.36:/work/exports/exported_link
bootcmd=bootp ; bootm
bootdelay=2
baudrate=38400
bootfile="/work/tftpboot/realview"
ethaddr=00:02:F7:00:19:17
ipaddr=10.1.77.77
gatewayip=10.1.77.1
serverip=10.1.77.36
stdin=serial
stdout=serial
stderr=serial
verify=n
ethact=SMC_RV-91111-0
```
* bootargs: we see that the kernel expects a nfsroot, and give 128M to the system.
* bootcmd: it tries to boot from network first (bootp). only after does it try to load from memory (bootm)

What we could do is to load manually the kernel into memory... so, how do we do?

u-boot seriously has issues in loading, damn...


**changing target**
Seems like it is possible to get a versatile-express Cortex A9 quadcore, from what I have seen in Codezero. That would help.
* https://wiki.linaro.org/PeterMaydell/QemuVersatileExpress

Therefore... yes, we switch back again to linaro. Phew.

**vexpress linaro**
* https://wiki.linaro.org/Platform/DevPlatform/Ubuntu/ImageInstallation
* making a kernel configured from linux linaro 3.6, with the ubuntu_vexpess_config file (disabled kexec though). using -kernel zImage to boot a kernel, and it seems to boot! vexpress_defconfig seems like a classic one where the rootfs is on nfs.
* To create the sd file used to boot (vecause the kernel expects to read in /dev/mmc), downloaded linaro-image-tools.
* Created a sd image with
```
./linaro-media-create --hwpack ~/Downloads/hwpack_linaro-vexpress_20121127-445_armhf_supported.tar.gz --dev vexpress-a9 -sd sd.img
```
* And the image can boot with our compiled kernel (from linaro as well) with the realview platform selected.
```
/opt/qemu-linaro-1.2.0/bin/qemu-system-arm -M vexpress-a9 -cpu cortex-a9 -m 1024 -serial stdio -smp 4 -kernel ../linux-linaro-3.6-2012.10/out/arch/arm/boot/zImage -sd sd.img
```

* It works relatively well, but booting is long, and needs time to close as well (although it will crash at the very end).
* Disabled DEBUG_RODATA
* Disabled CLCD
* Disabled boot logo

### 03/12
**versatile express booting**
It still takes time, about 100 seconds to have a prompt. Because it is waiting for the disk drive at / only?

Discovered a tool called bootchart. Surprisingly, qemu allows to apt-get, and it download things too! That is cool.

**vexpress memory structure**
0x60008000 is the starting address for the kernel. zImage is placed at 0x60010000 by qemu. Sadly, we don't have the reserved memblock interface (that was useful btw).
The memory goes up to 0xA0000000. Separation will be easy, 256M for each kernel (from 0x60000000 to 0x70000000). More fun if we can run more systems. We will 

**Step 1: initrd**
Must find where the initrd is copied. The debugfs exist within /sys/kernel/debug/. memblock/reserved reports:
```
   0: 0x60004000..0x60007fff
   1: 0x600081c0..0x60c7aaf7 << most probably kernel only
   2: 0x67ff8000..0x67ffcfff
   3: 0x67ffdfc0..0x67ffffff
```
But no real traces of the initrd. Will try to extract it from the sd card, and feed it to qemu. Maybe there is none?

### 13/12 
**disk waiting** 
The slow boot is troublesome. The message is 
```
The disk drive for / is not ready yet or not present
```
I suspect a potiential incorrect uuid associated for / . From http://liquidat.wordpress.com/2007/10/15/short-tip-get-uuid-of-hard-disks/, using
```
ls /dev/disk/by-uuid/
```
I identify that the uuid is effectively present both in the devices and in the /etc/fstab. The uuid corresponds to the disk labelled rootfs, so it is not the problem. Another way to do this is
```
blkid /dev/mmcblk0p2
```

Adding the debug param, but it did not really help.

I installed bootchart sometime ago, I thought it did not work. Seems like it does now, and files are in /var/log/upstart and /var/log/bootchart. Goota retrieve them.
```
sudo mount -o loop,offset=54525952 -t auto sd.img mnt/
```
I suddenly remember something wrong with the sd, about being very low in speed due to the writing method.
```
-drive if=sd,cache=writeback,file=sdcard.img
```
Although it should not be this problem, since we are not doing write intensive operations.

**config proc**
is present.

**initrd**
From the sd card, there is one. It gets loaded into 0x60d00000. But somehow, I think it is not the one we load originally, because the kernel asks for some modules (v3.6) absent from the initrd (v3.7). Checking the memblock/reserved, there is finally a memory reservation appearing.

Result, I think initrd does not affect the kernel; we can start it even without initrd.

**compile kernel to another location**
Trying to do that: changing the mem= to 0x7000, and changing Makefile.boot. I don't know how qemu loads it at that address, and I guessed it read from I don't know where the address of the kernel. That is because qemu knows the kernel is lodging at that address. Trying to change it by recompiling qemu?

File ./hw/vexpress.c
```
.loader_start = 0x60000000,
```
I think it has something to do with that. Changed its value, and it worked, and whats more, there is just no need to edit Makefile.boot, the AUTO_ZRELADDR does the job. Copied things by qemu include:
```
0x70000000-0x1c bootloader
0x70010000-0x40cxxx zImage
```
I am pretty sure bootloader is a simple jump to 0x70010000.

### 14/12
Applying patches, things work so far.

Adding kernelcore=128M with mem=256M (from 0x60 to 0x70), and we have 0x60 (kernel), 0x6d-0x6f (some reserved zone) blocked. Other memory is removable, and is actually lowmem... dunno if my thesis was wrong in this asumption that lowmem can not be migrated, which is actually not the case: as long as physical data located inside lowmem is referred as vmem, it is movable.

There is a bug in the series of patches for muxos, in the beginning. If muxos is not enabled, the condition
```
if (!muxos_init())
```
will fail. Remember to add ifdef.

### 17/12
**one more dependency in the files** 
About the parameters needed by SPARSEMEM, one of them is supposedly configured within mach/memory.h. Those parameters are actually contained by mach/memory.h, but included within asm/memory.h only if the architecture has CONFIG_NEED_MACH_MEMORY. This option has been included in later versions of the kernel, and need to be specified manually in the Kconfig of vexpress.

**address of muxos base**
60004000 is within a reserved region, and as such can not be used. Trying 0x9fff0000, which looks like within a unremovable region, but not reserved. The removing is successful but the ioremap fails, the location might be too ambitious. Trying 0x61000000 for now.

So, it fails at all ocasions, so I guess it might be a code incompatibility instead of a placement issue. I understand now: the location is considered ram, and therefore refuses to be mapped (got this problem before). memblock remove was supposedly able to remove from proc/iomem, but it does not look like the case anymore. Oh, i remember, that is because the system is supposed to scan each of the memblocks and add them into SystemRAM, but that is not the case right now (ie it add the whole range).

Alright, I did not remove them early enough, I need to go even earlier! The function building SystemRAM is located at request_standard_resources(), called quite early. Maybe we can use the .reserve like tuna?

Used it, and it cut the SystemRAM correctly, although ioremap continues to fail: this is because pfn_valid always return true, (there is no HAVE_ARCH_PFN_VALID), and is incidentally dependant of ARCH_HAS_HOLES_MEMORYMODEL. For now, I select it, but at term it should be a muxos requirement.

Managed to finally allocate MuxOS.

**slow start due to disk**
I still have no idea of why the start is SO slow. But i suspect it is due to the sd card emulation which is slow by itself. On boot, many things are likely to be loaded and run.

NFS might be a bit faster, trying this, but it has the exact same speed, and that is pretty lame...

Looked at the bootchart, and for example, the network service takes about 40 seconds to start. Maybe it can be worthwhile to disable it.

**load image** 
Seems like there are issues when the arguments have not been provided entirely.

**linux vanilla suspend** 
And there is no request_suspend_state, that was a android operation. Using pm_suspend instead, which is the direct call (no early suspend). Then trying to enable the suspend... but not sure they have it (and it is likely they don't). 

It is probably because there is no suspend_ops. Adding a basic one for now, but I guess I may have to implement the lower levels.

**echo mem**
is working at high levels. The mmc acts up though, making qemu crash. I suspect it is because it is ejected
```
mmc0: car 4567 removed
```
Guess I only have to make it static.

**TC2**
Vexpress has some power management things for the TC2 chip, a A15 based board. It is not compatible with the A9 board. There may be a need to port the hotplug code to it.

**hotplug cpu**
Can offline a core with
```
echo 0 > /sys/devices/system/cpu/cpu1/online
```
Possible in qemu. Although I don't know the exact state of the cpu yet...

**plan**
First, analyze the booting operation regarding the core management.  
The next step will be to try to boot up a system on core 2 and 3 instead of the 4 cores.  
Third, cutting down devices to the strict minimum, so there is a hope to run multiple instances (best would be to only need the cpu to be online, actually).  
Fourth, similar to what we have on muxos, it would be to load and run another system. Up to now, we can do this except suspending. This would require to unload a core and check if it is possible to hook it to the new kernel. Additionally, qemu has no reset commands, and its bootloader is a crappy one which only jumps to the zimage head (it does not handle image loading or whats not). 
Fifth, add some tweaks to allow/disallow hotplug between devices.

**booting operation**
check the file hw/boot_arm.c of qemu.

**qemu memory dump** 
```
x /fmt addr
```
or use xp for physical dump

### 19/12
**research axis**
Solve the memory security problem first. If it is possible to protect each OS from accessing the memory of another, it is golden as I broaden the usage model.  
Highlight what we can do with our work, that Cells/dual SIM/multi user can not do. It is a key condition.  

About parallel execution, it is a further way, and the biggest concern is the sharing of devices. It can be designed as a future work. Check whether it would be possible to run.


### week review
How to justify the need of multiple kernels:
+ A use case is for example power management. Kernels can be optimized for low power consumption over the course of the utilization (with appropriate governors or voltage settings), as we can see for many custom ROMs; but some kernel features can not be cut off totally, like wifi or other things. With our method, we can go even further in kernel optimization by cutting off the power of possibly unused parts, and switch between kernels (or start a new session) according to our needs. In the other case, tablets and others can be overclocked over short burst, and an appropriate kernel may do so.
+ Parallel task execution, but that would require to run kernels at the same time
+ Security concerns, although Cells achieves same functionality without the use of multiple kernels
- Multi userdata, but Android introduced this so...
+ Sandboxing, testing applications on various versions of Android on the same device (and without flashing things)
+ Gaming? some apps are tied to one system apparently, so we may have multiple characters
On a larger scale:
+ Multiple runtime environments is possible with others OSes, as long as they respect (or we enforce) the smae device state. 


Over Cells:
+ Multi-system: Provide a way to run multiple flavors of Android (MIUI, Cyanogen, etc...) and multiple kernels (because they are optimized differently). 
+ Less overhead, easier to implement and use on its current form
- RAM: Higher RAM usage
- Parallel task execution with device sharing

Over general flashing:
Using custom ROM is a matter of choice, using better supported distribution by community (kernel), and other "skins" giving another user experience (Android).
+ Current methods of supporting multiple Androids involve repetitive flashing and backups, via tools like Rom Manager, CWM, Titanium Backup, etc... With our solution, no risk of turning a smartphone into a brick, as there is no risk involved for the initial instance (no reflashing of the data/boot). Switching kernels and systems is just a matter of adding files into the SD partition. The initial kernel would need to be replaced though.



TrustZone :
Security on mobile devices has been a hot topic in the recent years, due to an increasing demand driven by manufacturers, service providers and end-consumers:
- the end-consumer handling transactions (like payments) or sensitive data (mails, corporate secrets) should be protected from any fraud and malware
- payable contents or feature should be protected from any fraudulent access and use

In their Cortex-A line of cores, ARM has introduced a security architecture in the name of TrustZone, allowing manufacturers to design SoC with security features enabled, and software developpers to make use of those security features. 
Two executions environment, the Secure world and the Normal (non-secure) world, coexist at the same time on the device: it could be for example a feature rich OS with a smaller secure OS. The hardware provides hardware barriers to separate entirely both worlds, so the Normal world can only make access to the Secure world via a monitor. Therefore, memory can be divided between both worlds, with a one-way access. 

- trustzone has been underused before the first half of 2012

Random ramblings:
- Each core has one specific context for the normal world and the secure world (ie 2 virtual cores). Switching between both requires to make a context switch.
- Both worlds are executed in a time-sliced fashion
- Each world memory is addressed in a 32 bit space, and a 33 bit indicates if it is the secure or the normal world
- Trustzone requires specific IP provided by ARM (a trustzone enabled cpu, Cortex A9 is one of them, bus, cache/memory controllers, GIC, etc...)
- Secure world is home of some independant library calls controlling specific devices (so code in the normal environment can not sniff those devices activity), or a full fledged OS.
- An idea would be to put a whole system stack within the secure world.
- The CPU starts in secure mode, and is probably put into unsecure mode at boot by the bootloader. I guess it flows like that: bootloader boots, put some code in memory (secure), switch back to unsecure, load the OS, and let the things roll. Every thing the OS requires to be in the secure world has been put beforehand. => this means we could have the main environment, which is secured, and start a secondary environment, which is unsecured.
- Each OS would have to lose previous usages of the secure world since it would be reserved to our usage.
- That would require to know how to tamper the MMU tables on switch-boot. It is however directly usable in MuxOS, as it would require to make a context switch on suspend.
- It is distinct from virtualization extensions, and only allows two environments (or at least, there is only one separation, but multiple environments can be hosted in one world). That is okay, because there is no really needs to run that many environments.
- In our usage, we don't need to allow communication between both systems, we only need the memory separation. (and to boot, memory separation is only one way)
- Switching is easy with a smc instruction if there is only two worlds. If there is a need to jump to one specific instance in one world, it would require first to jump to the world, then make the work we did before.
- this render two kernels inequal (ie can't use the same source code in theory)
- there is the monitor mode in between
- On Galaxy Nexus, it is not sure what is loaded in the secure world, but for sure, if we were to run the regular kernel in a secure world, it would execute smc instructions, but it would stay in the secure world
- One problem in using trustzone, it is he whole components that usually requires trustzone do not benefit from them anymore.
- From a code example, it seems the monitor is some code we are expected to implement.
- devices are memory mapped; protecting the devices requires to tag the memory address as secure or non-secure.

Implementation details: p1175
- there is one processor mode, monitor mode, that is expected to be the mode where bridge code between both worlds is executed. The monitor mode is coded as 10110.
- smc (from any world) jumps the cpu to monitor mode, not directly to the application processor mode (user, system, irq, etc...)
- we can read the security state of the executed code with the SCR.NS bit (1 = non secure)
- SCR can be changed only in secure state. Jumping to non secure is done with SCR.NS <= 1, then exception return
- general purpose registers are not banked, cpsr and spsr are not banked
- many coprocessors are banked, and the one that are not banked are global
- exceptions vectors are banked, but in our case it should be fairly safe
- non secure -> secure done via SMC are some exceptions
- secure -> non secure done via small mechanisms, which ones?

SCR: p1380
- accessible only in secure state + secure privileged modes only (no secure user mode) 
- reset value of 0
- NS is the bit 0. 0 = secure, 1 = non secure. When in monitor mode, the initial state of this bit is irrelevant.
MRC p15,0,<Rt>,c1,c1,0 ; Read CP15 Secure Configuration Register
MCR p15,0,<Rt>,c1,c1,0 ; Write CP15 Secure Configuration Register

SMC: p1203
- calling SMC within monitor mode stays in monitor mode
- in monitor mode entry, spsr and lr have the return value, so a return from exception is enough to jump back to non-secure world (forgot which one it is though)
- SMC called only in privileged mode (ie user app can not do this)
p1157
Monitor mode has access to both secure and non-secure copies of the register

Translation tables: p1283
in 1st level descriptors, there is a bit in the translation table entry indicating if the target section is secure or non secure.
in 2nd level descriptors, there is no such bit. 

Question is, what does this do in reality? 
If the bit can be changed anytime, looks like a bit useless to me

ttrb0 p1285 and p1387, is a banked register. So the previous bit looks only "indicative". The secure copy can be accessed if CP15SDISABLE is low

Secure and non-secure address space p1300
- There is two physical address space, one secure, one non-secure. registers containing the address of translation tables are banked between the two modes. 
- Secure translation entries translate to either non secure address space, or secure address space.
- Granularity of space is 1Mb.
- To protect data, that would require a system to map them??? I am a bit confused now. How is protected an address in memory ?

Memory region attributes p1306
"An attribute is checked by the hardware to ensure that a region of the memory that is designated as secure by the system hardware is not accessed by memory access tagged as non-secure." The hardware refers to the memory controller on the axi bus, which will do the verification and issue an abort exception (to verify) if attribute is not compliant
- Still, must check those attributes. From what i can see, there is two copies of the attribute table, therefore it is hard to see how one.
Requires a TrustZone Address Space Controller

Question is, they say there is two physical address space, one for Secure and one for non-secure. Therefore, physical memory is connected where? It is reasonnable to think that there is only one memory and it is accessible by both worlds

Memory Accesses description p1263
Could not find any description, but we can infer this:
Suppose an address is tagged as secure, and there is a cache line for this address marked as secure. If a non secure tlb is tampered to access that address, the cache line will not return its content and the system will be forced to look into the memory. The controller will deny access to the line because the secure-state is not the one expected.
This means the ns bit is only an indicator of the nature of the data we WISH to access, not of the real state of the data.

Execution flow: example
- system boots in secure mode
- system initialize the monitor (including setting a stack and all)
- system sets the monitor vector table, with one of the vectors (SVC or SMC) leading to the smc handler. Sadly, it can only be read and edited in secure mode
- system triggers smc
- monitor executes smc handler
- monitor saves the context
- monitor makes a jump to the normal world (could be the zimage)
- ns system boots ns OS


### first draft
Abstract: 
- A need: multiple environments on one mobile device
- From: Corporate needs and user pressure to use their own device
- Key requirements: security, separation
- Proposing lightweight alternative (no need of heavy software layer like virtualization or Cells), without performance overhead (run natively on hardware), and keeps native feature (no need of device virtualization yet) -> not true if we are forced to do device partitioning
- We go one step further and propose a memory sharing model (and a cpu sharing model)
- Made preliminary implementation with two linux/Android and some functional results

Introduction:
- Introduce the main need
- Other potential needs: multi-user (just introduced by Android 4.2), developping sandbox, changing user interface
- Existing MOBILE general kind of solutions, with virtualization (explain its drawbacks) and 
Cells (workspace virtualization), CodeZero (hypervisor based), MobileIron (server/client), Devide (server/client), OKL4 (but does not support multi-general purpose OSes?)

Architecture:
- Two modes of execution: Sequential and Parallel, affect CPU and device sharing 
- Sequential: 
- - Principle: At all times, one OS run, the other are paused
- - - CPU: each OS gets the most of the limited processing power. Security boost as execution time of each environment is tightly controlled (no risk of one environment hogghing resources, or monitoring), but impossible to run parallel tasks between environments.
- - - Devices: each OS gets the most of the devices functionality.
- - Implementation: Readily implementable with suspend-to-ram like, no problem of device sharing
- Parallel: 
- - Principle: Multiple OSes may run in parallel
- - - CPU: One machine has at least one static core. Other cores can be dynamically added to or transferred between environments. 
- - - Devices: Higher complexity as devices need to be shared. See NoHype
- - Implementation: Implementable with CPU hotplugging, and specific support from the devices

- Memory sharing: Model of dynamic memory layout across all OSes
- Regardless of if it is sequential or parallel, SMP hardware can/will ensure coherency of cache and memory. 
- No protection: readily implementable on ARM devices with memory hotplug like
- VE: most complete solution to preserve a "per environment" protection
- Trustzone: enough for a "per world" protection (secure/non-secure), need hardware/software support (and potentially bootloader cooperation). Contrary to x86 needs, this is a viable solution for mobile devices


Others: 
- NoHype has virtual i/o devices... do we need to provide this?
- We basically move hypervisor like functions into the OS -> OH YES, set the monitor within the OS space and we are all good yeah, no need of a separate code (remember, we just need to set the monitor location).
