# Daily notes on fastswitch
Roughly edited document research notes
Author: Alexis Hé

***

### ->15/11
Worked on thesis

### 16/11
**Continue refactoring code** 
global guidelines: add the debugfs first, to make sure the infrastructure is seen...

Various facts:
* printk is included with <linux/kernel.h> 
* we have our header in /muxos, and to make it accessible via an #include <file>, that would require to put /muxos in the "system dir list", OR put the header in the /include directory, it works as well 
* in git rebase, to edit a commit message only, use reword instead of edit+commit+rebase 
* #define pr_fmt(fmt) "prefix: " fmt, define it first in a file: make things so much easier! 
* A process to edit and include changes in previous commit: edit, stash, rebase, git stash apply, checkout files not involved, commit amend, rebase, stash pop 
* vi replace all: :%s/OLD/NEW/g 
* in a header file, a function is either static of extern. no keywords will add an implicit extern for functions. (not the case for variables)
* split the external api in one header, and internal functions across multiple files in another header

### 17/11 
**Various notes:** 
* put muxos_internal.h as a local file
* don't bother with a muxos repertory in include... we put it in linux.
* Maybe make one debugfs per file, with the relevant operations within the file? yes, that could a better solution instead of scattering the functions all over

**reimplementation step** 
First, the booting operation.
We first need to free memory in cascade. That means:
* (0) base address of the Muxpage, size (global.h)
* (0) helpers to block memory (any kind) (in core.c, global.h)
* (0') put init to its real location and the blocking memory functions in tuna
* TEST, check if memory of the muxpage has been removed well
* (1) muxpage global struct
* (1) function to retrieve the instance number p5 (core), add to init
* TEST, check if the muxpage can be read, and if the value has been set correctly
* (2) CST definition and struct (global.h), including AVAILABLE, OWNER_MASK
* (2) function to initialize the CST p5 (memory.c), add to init
* (2) the debugfs interface to print the CST (memory.c)
* (2') block memory for the CST, + set shared a few sections during initialization.
* TEST, verify the state of CST
* (3) add strict onlining and offlining section p3 (memory.c, internal.h)
* TEST, verify if section get offlined or onlined correctly with function
* (4) the debugfs interface to free memory, update CST, (don't add RST) p3-4(memory.c)
* TEST, verify that the function offlines the correct number
* (5) add the load_file tool (tool.c, internal.h)
* (5) add the debugfs interface + function to load whatever file and update the muxpage of target if it is kernel image and initrd p2 (core.c)
* TEST, check if all files are at the correct place


**reverse ssh** 
```
lab: ssh -R 5678:localhost:22 -p 4567 krosk@ip
sushe: ssh -X yuhe@localhost -p 5678
```
Don't forget to redirect the port 4567 in vbox

Simultaneously try on realview or exynos?

### 18/11 
* (6') get the atags related code up and working in a muxos_arm.c p6 p2 (may need to edit makefile).
* (6) debugfs to add them + arch-independant operations
TEST, check if they are properly written too
* (7) debugfs to issue the boot order and update the muxpages when loading (core.c)
* (7) code support in the suspend operation to change the flags p10
TEST, check if muxpage flags are edited
* (8') add the code to handle the switchboot order, MMU disable, but no platform reset, p7 p8
TEST, if the special code gets executed only on a switch-boot order
* (9) add the header modification for arm p1 (head.S)
* (9') add the reset code (sleep.S)
TEST, if the second instance boots on reset
* (10) add the code to add the RST, read the RST in init + update cst p5 (cst.c)
TEST, check if the sections get updated correctly
* (11') add the code to reload the backup (core.c / arm.c)
* (11) add the code to handle the suspendswitch order p7 p8 (suspend.c)
TEST, check if the suspend suceeds or not

### 19/11
**Administration**
* the advisor must be set by the jiaoxueban, not the student
* mails sent by administration were in the spam...
* the result of the anti-plagiarism check is on the advisor

**observation on linux header structure**
* we need to define specific values for one board, it is arch dependant and therefore must be put in mach directories.
* the include directories work by layer, one on top of another. the generic layer, then the arch layer, then the plat/mach layer.
* On ARM, <asm/...h> will point to arch/arm/include/asm/. <mach/...h> will point to arch/arm/machxxx/include/mach/.
* we decide to put it in a file called <mach/muxos.h>, which is the deepest we can go. plat could work as well, but mach is one step deeper.
* if no muxos.h has been placed on a target, then it will not work.

**reimplementation**
* 0 to 3
* for the free_memboot command, an attempt to set a section claimed, but not orphan

**meet again a problem in freeboot**
* offlining several sections in a row via the freeboot will trigger a page error

**check the following things:**
* reduce the timeout (no)
* print the values directly in remove memory (u64) (done)
* offline one by one all sections, in both order to see if it is a static problem (yes)
* offline via a hard-coded sequence

### 21/11
**observations on offline impossibility**
* The bug happens at section 0x9400-0x9500 or 0x9500-0x9600, when all sections below have been freed.
* It has nothing to do with free_memboot, it happens as well when sectiosn are manually disabled
* It has nothing to do with the offline timeout
* Bug happens after a certain number of sections have been offlined (5 to 6), regardless of the order, even if it is not consecutive.

This may mean that the memory hotplug has not been implemented well?

```
[   82.805847] memory hotplug: 95000 - 96000
[   82.837554] kernel BUG at /home/yuhe/dev/android/omap_base/mm/page_alloc.c:873!
[   82.852478] Unable to handle kernel NULL pointer dereference at virtual address 00000000
[   82.875762] pgd = c1a14000
[   82.886199] [00000000] *pgd=00000000
[   82.897644] Internal error: Oops: 805 [#1] PREEMPT SMP
[   82.910644] Modules linked in:
[   82.910675] CPU: 1    Not tainted  (3.0.31-ge29f985-dirty #80)
[   82.910705] PC is at __bug+0x24/0x30
[   82.910705] LR is at console_unlock+0x1a0/0x1d4
[   82.910736] pc : [<c005b55c>]    lr : [<c00a7be0>]    psr: 60000093
[   82.910736] sp : c1861be0  ip : 60000093  fp : c1861bec
[   82.910736] r10: c086f3c4  r9 : c0e24000  r8 : 00000000
[   82.910766] r7 : c0e24018  r6 : 00000002  r5 : c086eba0  r4 : 00000320
[   82.910766] r3 : 00000000  r2 : c083e760  r1 : 60000093  r0 : 00000059
[   82.910766] Flags: nZCv  IRQs off  FIQs on  Mode SVC_32  ISA ARM  Segment user
[   82.910797] Control: 10c5387d  Table: 81a1404a  DAC: 00000015
```

```
[   82.912994] [<c005b538>] (__bug+0x0/0x30) from [<c011b93c>] (move_freepages_block+0x164/0x170)
[   82.913024] [<c011b7d8>] (move_freepages_block+0x0/0x170) from [<c011d380>] (__rmqueue+0x338/0x510)
[   82.913024] [<c011d048>] (__rmqueue+0x0/0x510) from [<c011e1a4>] (get_page_from_freelist+0x364/0x524)
[   82.913055] [<c011de40>] (get_page_from_freelist+0x0/0x524) from [<c011ec0c>] (__alloc_pages_nodemask+0x104/0x7d4)
[   82.913085] [<c011eb08>] (__alloc_pages_nodemask+0x0/0x7d4) from [<c014d310>] (hotremove_migrate_alloc+0x24/0x2c)
[   82.913116] [<c014d2ec>] (hotremove_migrate_alloc+0x0/0x2c) from [<c014e818>] (migrate_pages+0xdc/0x408)
[   82.913146] [<c014e73c>] (migrate_pages+0x0/0x408) from [<c05f95f0>] (offline_pages.constprop.16+0x4b4/0x5dc)
[   82.913146] [<c05f913c>] (offline_pages.constprop.16+0x0/0x5dc) from [<c014d53c>] (remove_memory+0x40/0x44)
[   82.913177] [<c014d4fc>] (remove_memory+0x0/0x44) from [<c0316914>] (memory_block_action+0xc8/0x168)
[   82.913208]  r5:c08c1600 r4:00095000
[   82.913208] [<c031684c>] (memory_block_action+0x0/0x168) from [<c0316a78>] (store_mem_state+0xc4/0x100)
[   82.913238]  r6:c78673f4 r5:00000008 r4:c78673d0
[   82.913269] [<c03169b4>] (store_mem_state+0x0/0x100) from [<c030a5ac>] (sysdev_store+0x20/0x2c)
[   82.913269]  r8:c0640420 r7:c78673fc r6:00000008 r5:c7866e10 r4:c58e57c0
[   82.913299] r3:00000008
[   82.913299] [<c030a58c>] (sysdev_store+0x0/0x2c) from [<c01a36ac>] (sysfs_write_file+0x104/0x184)
[   82.913330] [<c01a35a8>] (sysfs_write_file+0x0/0x184) from [<c015101c>] (vfs_write+0xb0/0x140)
[   82.913360] [<c0150f6c>] (vfs_write+0x0/0x140) from [<c015129c>] (sys_write+0x44/0x70)
[   82.913360]  r8:00000000 r7:00000004 r6:00000008 r5:40fbd784 r4:c60b5d20
[   82.913391] [<c0151258>] (sys_write+0x0/0x70) from [<c0057480>] (ret_fast_syscall+0x0/0x30)
```

After some source inspection, I remember this bug: it is a bug with HOLES_IN_ZONE, encountered the 30/08. The issue is that the kernel checks whether the start of the range we free is in the same zone of the end of the range. It happened before for a certain kernel.

Anyway, HOLES_IN_ZONE must be enabled.

**right margin in edition** 
Linux has a coding style of a 80 column margin. Best is to respect this margin. 
* In vi, use:
```
:set colorcolumn=80
```

**reimplementation**
* 4, 5, 6, 7

**thoughts about atags**
Muxos expose only and only one recipient, in the muxos page. A temporary recipient, if needed, is the initiative of the architecture. For arm, it is necessary because the atags are not available so soon.

Muxos will call a backup(size, location) where a arch dependant function will fill the muxos page recipient.

For arch dependant code, we use muxos_$(ARCH).c.

### 22/11
**Reimplementation** 
* 8, 9

**New guest process (10) **
* create userdata.img (size does not matter) with 
```
make_ext4fs -l 256M -a data new_userdata.img mnt/
```
and push it to /sdcard/
* if needed, a system.img must be pushed: one downloaded directly will not work, you have to process it with simg2img.
* same for a cache.img

* get a ramdisk and extract it via
```
gunzip -c ../boot.img-ramdisk.cpio.gz | cpio -i
```
* create parent folders in init.tuna.rc, and mount the relevant file system (userdata, cache)
```
mkdir /parent 0700 root root
mkdir /parent/data 0700 root root
mount ext4 /dev/block/platform/omap/omap_hsmmc.0/by-name/userdata /parent/data wait
mount ext4 loop@/parent/data/media/userdata.img /data wait noatime nosuid nodev nomblk_io_submit,errors=panic
```
* For jb, fstab.tuna does not support loop mounting, so we use the full line instead:
mount ext4 loop@/parent/data/media/system_jb.img /system wait ro
* repack the ramdisk via
```
find . | cpio -o -H newc | gzip > ../boot.img-ramdisk-repack.cpio.gz
```

* get a copy of the atags with
```
cat /d/muxos/boot_params > /sdcard/atags
```

* the load_image procedure goes like this: kernel to 0x91808000, initrd to 0x92800000, atags to 0x90000100
* the kernel must be recompiled with
```
mem=768M@0x90000000 initrd=0x92800000,0x4f800,0x92800000
```
and **Makefile.boot** must be edited

**compressed image header**
* curious bug. A cmp instruction hangs. Register r5 with the flag value (0xDEADBEEF, but it is regardless of the value).
```
  	ldr	r4, =blank_value	@ r4 = CLEAR VALUE, for reset
		ldr	r5, =flag_value
		ldr	r6, [r3, #MUXOS_FLAG]	@ r6 = trigger flag
		str	r4, [r3, #MUXOS_FLAG]	@ reset flag to avoid jumps
		cmp	r0, r5
		strne	r5, [r0]
		bne	skip_switch		@ skip if not triggered
```
Regardless of the value of r5, it hangs.

After piinpointing, it is just the comparison of this value at this place at this time... replaced the instruction with subs instead, so the flags get updated correctly.


**early printk**
* JB has corrected the early printk, so there is no need to comment the lines in kernel/printk.c and arch/arm/kernel/setup.c (or smth like that).

**jb 2nd kernel**
Hangs somewhere... but I have no feedback at all. It is frustrating... The header gets executed, but no way to know up to where.

Fastswitch works well with the same initrd and atags. They are not responsible... might be the file copy, or the memory remove.


### 23/11
**Changing approach**
We will change completely the approach of the reimplementation: the first step will be to make the switching system functional first before thinking of the CST. Again, we do this step by step.

This is due to the fact that the most unstable and hard element is wheter we can run a second kernel at a different place, the rest (memory sections etc...) can be easy to debug.

Steps are to first make sure the initial system can run with SPARSEMEM and online offline a few sections, then return to FLATMEM.

So far, we more or less re-applied the patches in order, up to the point where we need to remove memory for the base page. One culprit is ARCH_POPULATES_NODE_MAP which must not be activated when there is no SPARSEMEM (obviously).

SPARSMEM is not the cause of the hang. Might be a logic problem...

The absence of initrd does not seems to affect the boot early, this is not the cause. I really think it is a memory related case.

**resolving**
Thought of a way to test the logic of the header: instead of jumping to the second kernel, jump to the same kernel by setting up the second target to be the current one.

Logic has no issues, jumping to self works.

Finally! got it working (on flatmem) ... DAMN, where was the problem? I have no idea!

It may have been the script to load the atags, forgot to specify a good address. Maybe.

seems like it works if the fsa driver and debugll are opened. Must be that if debugll is actived, debug-macro may need to HAVE values or it hangs??? Seems like the case! So be careful in the future, if debugll is enabled, debug-macro must be configured.

### 24/11
**zhang's problem**
He has a MMU problem where he can not manage to sub the pc to match the physical address and the virtual address. After some investigation, might be because a branch prediction is performed without taking into account the changes in the translation table. According to the TRM, enabling or disabling the MMU, or making changes in the translation tables, require to invalidate entries in the branch prediction.

According to the technical reference manual (TRM), the core performs branch prediction before reaching the actual branch. That is, in our case, when you do sub pc, r3, it has computed the destination before you changed the value in the translation table, and so it wil jump to 0 instead of 0xc000****.

From the TRM: 
The branch predictor maintenance operations must be used to invalidate entries in the branch predictor after
any of the following events:
• enabling or disabling the MMU
• writing new data to instruction locations
• writing new mappings to the translation tables
• changes to the TTBR0, TTBR1, or TTBCR registers, unless accompanied by a change to the
ContextID or the FCSE ProcessID.

I was careless before, and only used a dmb to make sure the translation table was updated. Somehow, it worked, but it may not work at all cases. So you might want to try this instead:

```
MCR p15, 0, Rt, c7, c5, 6 @invalidate the branch predictions
isb @ let it take effect
sub pc, r3
```

**implementation strategy**
Make the swith working with static mappings first. To test the framework, we can do an auto-switch to self.


The sleep4xxx.S has some platform independant code, might be a good idea to put them separately. We could do a function with an address as a parameter, which is reponsible for updating the mmu transtable before disabling it. But it is not worth the trouble.

Below is the procedure to update a transtable entry, as told by the TRM.
```
typical code for writing a translation table entry, covering changes to the instruction or data
mappings in a uniprocessor system is:
STR rx, [Translation table entry]
; write new entry to the translation table
Clean cache line [Translation table entry] : This operation is not required with the
; Multiprocessing Extensions.
DSB
; ensures visibility of the data cleaned from the D Cache
Invalidate TLB entry by MVA (and ASID if non-global) [page address]
Invalidate BTC
DSB
; ensure completion of the Invalidate TLB operation
ISB
; ensure table changes visible to instruction fetch
```

**clean cache line**
While not obligatory in a monoprocessor (because only one core check the cache), it must be a "clean data cache entry operation" at the point of unification (PoU). The operation will be 
```
mcr p15, 0, rt, c7, c11, 1 @ rt is translation table entry address
```

**invalidate tlb by MVA**
The same line must go through "invalidate instruction cache by address" at the PoU. It is unclear what MVA should be (page address they say, but I don't know if it needs to be aligned (not cache line aligned they say).
```
mcr p15, 0, rt, c7, c5, 1 @ rt is mva to affect
```

**invalidate btc**
BTC stands for branch target cache, and as such requires invalidation by the following operation:
```
MCR p15, 0, rt, c7, c5, 6 @ rt is mva to affect
```

**Invalidate both**
This one will invalidate all instruction cache and flush the btc.
```
mcr p15, 0, rt, c7, c5, 0
```


**Compare memory**
In order to autodetect the memory ranges overwritten by the bootloader, I decide to implement the function with the kernel.

During the processus, I stumbled upon a weird bug. I use .equ to define constants, but some values make the platform unable to start (even without flashing).
```
.equ same, 0xFFFEFFFE
```
will block, while
```
.equ same, 0xFFFFFFFE
```
will work.

Changed method; Observed that between 0x80000000 and 0x88000000, there are 0x6f96b0 of data that has been erased.

Final method: we make a copy of the system to a target address just before a reset command. After the reset, we make another copy at the head of zImage. We compare both copies with a machine state. Basically, we make a list of memory ranges that are different, and placed in the muxpages. Then, on boot, we process the list and print a list of memory ranges to exclude. Should work.


**Next**
* add the switch command
* add the switch pre-suspend operations, including backups and the jump to resume (p8)
* add the tuna mrr and resume address (p6)


### 25/11
**reimplementation**
Done: suspend command, pre-suspend operations, post-suspend operations, helpers to save resume address and mrr, suspend operation

**silly bug of pointers**
A curious situation: I have a struct defined like this:
```
struct page {
	unsigned int * regs;
}
```
The regs pointer is afterall just a pointer to the beginning of a zone that has been previously mapped. I want to access it with:
```
struct page foo;
foo->regs[0]; // the element at address regs
foo->regs[0x10]; // the element at address (regs + 0x40)
```
Curiously, it returns (foo->regs)[0] instead of foo->(regs[0]). i.e. if regs is not initialized (= 0), foo->regs[0] makes an access to address 0, and foo->regs[0x10] makes an access to address 0x10*sizeof(unsigned int).

Defining regs like this works better:
```
struct page {
	unsigned int regs[];
}
```

**new operation framework**
It has its perks, by the fact that everything is accessible from muxos_core.c. In the suspend operation, there was a small problem: how to know whether a suspend suceeded or not? In the old version, I relied on an error return value. Here, I did not export the error value so I don' know if it managed to work or not. However, we can rely on the allow_core flag to be disabled if the suspend did work. 

Second, I was forced to disable the allow_core during the suspend operation in the previous version, because I was expecting to jump back to the second resume operation where I would read the flag. In our case, since we do everything at the end operation, there is no more absolute need to disable the flag (i.e. once I jumped, I let the natural course happen). I disable it to indicate "we processed the flag, okay".


**Exit an instance**
* Reintegred the code for this, but wakelocks make it impossible to suspend with the screen opened.
* Reimplemented the wakelock skip, but it is often unstable (immediate wakeup -> power-off or reboot). A way to correct this would be to yield (?).
* There is a problem with the PowerManagerService, which is always active at the end.
* While the secondary instance can shut down, the main one will not be able to.


**Next** 
CST and memory sections

### 26/11
**reimplementation** 
* repatched CST, free_memboot, RST.

**Next**
* exit reclaim, button switch

### 27/11
Reintegrated the exit claim, as well as correcting some forgotten bugs (like CST not updated when a second instance boots).

**smc bug**
Just so I don't forget it, there is a nasty bug preventing the platform to quit properly: the SMC zone is shared between both kernels, and when booting the seocnd instance, it gets rewritten with teh second instance data. That makes thinks complicated when the first instance wants to power up (as it is likely to execute code in the SMC zone), or just don't allow the second kernel to power off...

**input button magic**
Many things to see.
* getevent is a tool which pools the /dev/input/ interface. pushing and releasing a button will triger a message.
* source: http://www.netmite.com/android/mydroid/system/core/toolbox/getevent.c
* What we could do is to remake a similar program, which will instead launch our muxos commands. It could be launched on boot. The drawback is it is a userspace program.

**implementation**
* Took the program getevent, removed unnecessary code, kept the strict minimum into it to poll events coming from /dev/input/event2. It is not strictly a poll: the poll() call stops and wait for an i/o to happen. Added a small machine state to handle a sequence of input, and trigger the command_switch according to the sequence.
* Second step is to automaticcaly launch this user space program on boot: edit init.tuna.rc and add a service as a main. The program will be launched from /system/bin/gnexmuxos.
* Third is to handle the delay in the suspend request. When pressing the buttons in the menu (with screen on), there is a delay forcing the platform to wait for wakelocks to expire, then jump. Planned to use enter_state instead of request_suspend_state, but wakelocks are still active. Tried to disable wakelocks, but there are stability issues.
* Best is to use request_suspend_state on the lockscreen, this is the most stable and the quickest.

With repetitive switchs, it does not seems to be very stable... Sometimes, the new system will hang when resuming (just after a [switch] message, so I guess it is around the assembly level.

**Next**
We don't do the memory transfer. There is few interesting things to code there, and it won't be used in the demo.

**Linaro**
* kernel source (not identified yet) http://android.git.linaro.org/gitweb?p=kernel/omap.git;a=summary
* image : http://releases.linaro.org/12.10/android/images/galaxynexus-jb-gcc47-aosp-blob

### 28/11
Noticed the existence of "keycodes" in the android init.rc scripts, which may be able to trigger a service on some key pressing.

**initrd size**
* tried to push a linaro ramdisk, and stumbled upon the problem of initrd size. The kernel does not boot when I report the size of the initrd in the boot command (which to my knwledge is where the size is read). 
* tried to load a size augmented classic initrd, no avail.
* The problem can not be solved easily as I can't read at all the logs. Can't see early printk with new initrd (but can see with old).
* When removing the ramdisk load, it performs okay though, I can at least see early logs up to the reboot. 
* Found the reason: ATAG_RAMDISK need to be tampered.
* Obviously, since I used my own kernel, it did not manage to run up to Android, but at least the initrd was unpacked.
* Got this error
```
init: untracked pid 305 exited
/home/yuhe/dev/android/omap_base/drivers/misc/inv_mpu/mldl_cfg.c|inv_mpu_get_slave_config|1792 returning 4
```
. I suspect this
```
init: cannot find '/vendor/bin/pvrsrvctl', disabling 'pvrsrvctl'
```
is the problem.

**linaro system.img**
* They don't have proprietary binaries: indeed, the file pvrsrvctl was absent. from the system.img
* After comparing the two system.img, discovered that there is quite a lot of differences. Linaro don't use them, therefore their kernel must have been configured to not use them?
* From the script provided by linaro here
```
http://android.git.linaro.org/gitweb?p=device/samsung/tuna.git;a=blob_plain;f=merge-gnexus-blobs;hb=linaro_android_4.1.1
```
, I copied the missing files (except fRom?). Still don't work.
* The error being within init, I inspected logcat and see:
```
load_driver(/vendor/lib/egl/libEGL_POWERVR_SGX540_120.so): Cannot load library: 
link_image[1891]:   345 could not load needed library 'libIMGegl.so' for 'libEGL_POWE... (too long)
```
It is indeed a missing file, again.
* The reason was meld not copying .so files!

Finally, Linaro has successfully run as a 4.1.2 Android (but is actually teh same kernel...). 

Worked, but if I add the gnexmuxos, it seems to not be able to complete the loading operation.

Linaro = same kernel

**Linaro small issues**
* It is very sensible to the size of initrd! remove any ~ files in it.
* remove service bootanim in the init.rc script, and it works like a charm (no more memory hog).



**Chroot Ubuntt**
chroot. This is how they launch ubuntu on android.

After some reading, it is intended to set one folder to be the root directory, so all subsequents applications running in chroot can view only up to that directory. 
Seems like a in-kernel mechanism, where we have on and only one kernel for all distributions. 
chroot environments are said to be "jailed", as they cannot access upper echelons. 

Ubuntu, after all, is strictly a distribution over a linux kernel, so with the right set of binaries it can run over the same kernel as an android.

**webOS**
* http://www.webosnation.com/webos-ports-posts-instructions-alpha-galaxy-nexus-open-webos-port
But they don't really provide images. One still has to compile it alone... Much trouble, he.

**Cyanogenmod**
* They have their own kernel and provide images, so they can be a good start. The only caveat is that the configs are inside the kernel and you get them only if you run one. Since I don't really want to delete my partition...
* http://wiki.cyanogenmod.org/wiki/Building_Kernel_from_source
* http://get.cm/?device=maguro&type=stable

Official instruction at: 
```
http://wiki.cyanogenmod.org/wiki/Galaxy_Nexus_(GSM)
```
They require CWM or ROM manager, using the update.zip, which erases the current /system. We don't want that...

There is no manual installation.

A link about how to make a update.zip:
* http://forum.xda-developers.com/showthread.php?t=1633025
* Basically, there are a set of files which are going to be copied, and a META-INF folder which contains a binary + a script. The script contains several commands like "mount", set permission, etc. We can do this manually...

Small problem: it is long to do manually, and symbolic links can not be created to a non-ecistant file (it is called a dangling link).

-----

**Next step**
qemu-linaro among the one of interests supports : 
* Exynos4210
* ARM Versatile Express (several cores)
* ARM RealView Platform Baseboard / Emulation Baseboard

See supports (defconfigs): http://git.linaro.org/gitweb for linaro kernel.

**Exynos 4210**
* http://en.wikipedia.org/wiki/Exynos_(system_on_chip)
* A dual core architecture, powering recent smartphones like GS2. The linux kernel is limited to two CPU only.

**Realview**
Realview seems to be more or less supported in linux kernel all recent versions (ie no diffs in history for the mach folder). 
However, there is quite a difference for the other folders, like kernel or common. -> we take the linaro one.
It has 4 cores max, and is a ARMv6 architecture... ok.

**Good reading material**
http://www.cnx-software.com/

### 29/11
**Cyanogenmod : run a rom without erasing system**
Summary of the problem: we have only a update.zip, which is supposed to replace the content of the system partition. What I did is to open the update.zip, understand the updater-script, and rebuild a proper cmsystem.img according to the instructions (which is usually format + copy system content + set permissions). I wrote a prog to do it in my place. Then, since we have a proper boot.img with the ramdisk, I edited the init.runa.rc to load the cmsystem.img and the cmuserdata.img instead (which we got from simg2img). flashboot boot and we are ready to go. This is how 

An automatic process can be done to build a system.img from a update.zip, althought booting it would require to have an altered ramdisk.

**CM10 config**
* http://wiki.cyanogenmod.org/wiki/Building_Kernel_from_source 
It says either to pull condig.gz from the /proc directory (doesn't work) or use a nice script they embed in their kernel:
```
scripts/extract-ikconfig boot.img > .config
```
THAT is sensible, thanks, but it did not work... damn. Might be because I downloaded a stable version.

**CM10 kernel**
Obviously, their kernel source have no trace of the tuna board... Damn. And it is absolutely not a uptodate kernel (2.6.xx). So maye they just use the official one?

Let's try our kernel! Although it defeats the purpose of our experience...

Had a small inconvenience, as the initrd size is bigger in CyanogenMOD. Here, it was okay to write the size as 0x50400 (real size being 0x500CE), but keep the atag to 0x4f800. That was not the case for the linaro... Anyway, seems like that the kernel works good, and that we can upgrade it to a secondary system.

Therefore, we have our unoptimiwed kernel instead of theirs. Oh well, whatever, it is their fault on this one, and they probably have no kernel from the beginning.

Result: CM = same kernel...


**Switch-bug**
Occasionally, I can't switch back. I think this is due to the fact that sometimes, debugll is opened in one but not in the other. I will try to disable it everywhere. 

It is totally this.

**gnexmuxos location**
Can't put an executable bin in /data/media, sadly. Put in it /data/local then... 

**Demo**
Seems like CM10 is not stable. I think this is due to the fact that the DSS stays up sometimes on suspend of initial instance. Why does it stay up? I think it is because I press power buttons. I am almost sure it is the DSS...

Changing tactics in gnexmuxos:
* press down, press power, release power -> switch
* or, even better: press power, release power, press down/up. This has the advantage of putting the device into sleep mode. NO that is a bad idea, because the full android goes to sleep, which requires us to press the button again. Not good.
* press power, press down/up, release power -> switch. Release the button only when the system switched. We do this for now, because it helps to make sure the DSS is closed.

### 30/11
**MIUI**
* http://miuiandroid.com/community/threads/2-11-23.18827/
* There is some odd command in the updater-script, called retouch_binaries. AFAIK, they have been removed in recent OTA updates.
* Without surprises, it runs. I don't like very much the MIUI... And it is heavy on RAM.

**linaro kernel**
is unusable. Their patches are not compatible with ARMv6 platforms ...
```
Error: selected processor does not support ARM mode `bfc r0,#24,#8'
```
Therefore, we may need to use the vanilla kernel.

**git clone ONE branch**
```
git init
git remote add -t $BRANCH -f origin $REMOTE_REPO
git checkout $BRANCH
```

**setting up environment**
Gotta use a nano environment, I think, but they only have it for the versatile expres environment. So maybe I just have to dump it altogether? Try it anyway. Best would be to run as well a u-boot for those platforms... And eventually rebuild a rootfs with a busybox.

**where to find resources for realview-mpcore**
Directly form ARM, of course!
* http://infocenter.arm.com/help/index.jsp?topic=/com.arm.doc.faqs/ka4134.html
* They provide U-boot image and Linux kernel image.
* The kernel is patched against 3.3 (patch included), and the config is also present (so we can recompile it ourselves). 
* The bootloader has been compiled from http://www.linux-arm.org/git?p=ael.git;a=blob_plain;f=u-boot/src/u-boot-armdev_090728_armdevCS_20100820.tgz;hb=HEAD
* Filesystem images are available via a link and point to Linaro...

* Download the Linux 3.3.0 kernel git. using images atm.

**realview-eb-mpcore**
What are the specs of this platform?
* http://infocenter.arm.com/help/topic/com.arm.doc.dui0303e/
* http://infocenter.arm.com/help/index.jsp?topic=/com.arm.doc.dui0303e/index.html

* Identified the u-boot branch at http://www.linux-arm.org/git?p=u-boot-armdev.git;a=tree;h=refs/heads/090728_armdevCS;hb=refs/heads/090728_armdevCS
* 

**Running realview step by step**
* Get both images ready

```
/opt/qemu-linaro-1.2.0/bin/qemu-system-arm -M realview-eb-mpcore -serial stdio -m 128 -kernel u-boot_bin_u-boot_realview.axf
```
will load something, but qemu will complain with the ethernet.

```
/opt/qemu-linaro-1.2.0/bin/qemu-img create -f raw sd.img 128M
```
will create an image

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
But no real traces of the initrd. Will try to extract it from the sd card, and feed it to qemu.

