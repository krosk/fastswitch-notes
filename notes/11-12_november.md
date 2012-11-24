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


