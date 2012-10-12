# Daily notes on fastswitch
Roughly edited documentresearch notes
Author: Alexis Hé

***

### 20/08
**Imported changes from Springnote**  
What a load of crap... Finding a good replacement for springnote is HARD.

**Reorganize patch data**  
Encountered a bug when SPARSEMEM can not run at all.
This is due to memory incompatibility between SPARSEMEM and a kernel only taking half of the memory. SPARSEMEM needs references up to the end of kernel. REMEMBER THIS.

Encountered another bug where second kernel addressed a bad location during the boot. I had a bug in the CONFIG_ARM_PATCH_PHYS_VIRT section of sleep44xx.S (inside the suspend operation), added a TO in the name.

Regression test works, phew...

Managed to upload (finally) the kernel into github.

Notice : I know now why the suspend section is used during boot: CPU1 going idle is going through this line.

Gotta enable ARCH_POPULATES_NODE_MAP, fast.


### 22/08
**Road map for enabling ARCH_POPULATES_NODE_MAP**  
This feature is used to initialize memory, including how the memory is mapped and where are the holes.
It does so by registering each active zone with add_active_range(start_pfn, end_pfn). then validate the whole thing with free_area_init_nodes(max_zone_pfns).

**Boot sequence on x86**  
```c
setup_arch() in arch/x86/kernel/setup.c
    initmem_init() in arch/x86/mm/init_32.c
        memblock_x86_register_active_regions(0, 0, highend_pfn) in x86/mm/memblock.c
            for_each_memblock do some checks of boundaries and values, and if correct do add_active_range
            is registering all memblock it can between 0 and highmem_end (or low_pfn_end). That is to say, the whole memory.
        sparse_memory_present_with_active_regions(0)
            does the job at calling memory_present for each region found
    paging_init() in x86/mm/init_32.c
        page table initialization work
        kmap_init()
        sparse_init()
        zone_sizes_init()
            determines the boundaries of each zone, then calls
            free_area_init_nodes(max_zone_pfn)
```

**Boot sequence on ARM**  
```c
setup_arch() in arch/arm/kernel/setup.c
    arm_memblock_init() in arch/arm/mm/init.c
        is executed at the very beginning of the kernel boot
        memblock_init()
        erforms memblock regiestering and initrd placement
    paging_init(mdesc) in arm/mm/mmu.c
        page table initialization work
        kmap_init()
        bootmem_init() in arm/mm/init.c
            arm_memory_present()
                for each memblock do memory_present (this is where we can substitute to add_active_range, and do sparse_memory_present_with_active_region after)
            sparse_init()
            arm_bootmem_free()
                is seemingly doing big job by preparing zone_size[] and zholes_size[]
                free_area_init_node() is where we can substitute free_area_init_nodes
```

**Porting**  
It will be hard to implement this, as several code sections have alternative route when not using ARCH_POPULATES_NODE_MAP.

```
in mm/page_alloc.c
    Two functions, zone_spanned_pages_in_node() and zone_absent_pages_in_node() are defined when not ARCH_POPULATES_NDOE_MAP.
    Both functions are redefined when ARCH_POPULATES_NODE_MAP.
    The whole ARCH_POPULATES_NODE_MAP API is also defined.
    alloc_node_mem_map() also use one more path.
```

In the current execution path, none of the alternative routes are used. I guess some of them are multi node, that is why.  
Therefore, the reason why it does not work when we activate ARCH_POPULATES_NODE_MAP must be because of the two redefined functions. In particular, zone_spanned_pages_in_node() is used many times by other functions, after the boot initialization. This should explain the crash.

By keeping the previous definition of zone_absent/spanned_pages_in_node(), we can enable ARCH_POPULATES_NODE_MAP temporarily without affecting functionality.  
    add_active_range + sparse_memory_present_with_active_region works.
Once active ranges have been added,  
and the definition of max_zone_pfns (copied from x86) is proper,  
and the two functions have been redefined,  
free_area_init_nodes can be used instead of arm_bootmem_free.  

It works, it looks clean.

At boot :
> movablecore option does not pose problems under 128M.  
> kernelcore option does not pose problems at 256M upwards. Many pages are free to be moved, in theory.
I managed to free a number of sections, there is one though where a bug happened (hangs then reboot due to lock), and is located at mm/page_alloc.c 873. To investigate.

There is also the problem of sparsemem being able to memmap the b3a0X range. This causes a problem as it is taken into account when counting movable zones.

**Restoring the ram console to default address**  
This cause some issues when we want to remove the block, but it is okay. It would be preferable later to move it under a safe location in-kernel or far in memory, it 'may' work.

**Setting environment on Oddity**  
I can upload a proper kernel from now on

### 24/08
**Meeting with Chen**

### 28/08
**wpa_supplicant of GNex**  
Nexus does not want to connect to my adhoc. This is probably due to a missing feature from the supplied version of the GNex wpa_supplicant. Binaries found in the web are all compiled with the wext driver, while the GNex has nl80211 driver, making them incompatible.

**wifi hotspot**  
Tried Virtual Wifi Router and Connectify, both failed. The reason being the wifi card I have, Intel 3945ABG, does not support "Hosted network" (virtualization?). Replaced this one by a Broadcom 4311 known as Dell 1490. Notice that HP bios has a "whitelist", so only a specific set of model is valid.

### 29/08
**stashing**  
http://git-scm.com/book/en/Git-Tools-Stashing

To save work in progress :  

    git stash

To restore :  

    git stash apply


**movable testing**  
It is better to use kernelcore instead of movable core. Memory higher than 0xb3a0x is protected so the kernel won't touch it. This memory must be "System RAM", so it will count towards the movable zone, although it is not usable, falsing the movablecore option.

With kernelcore = 256M (vmalloc = 256M), the range that is locked to kernel is 8f00X to 9800X included. This is slightly inconvenient.

With kernelcore = 256M (vmalloc = 768M), the range that is locked to kernel is 8200X to 8f00X included. This is far much better. <<< **Setting to keep using**

This is due to the fact that the kernelcore will start allocating fixed memory at the end of lowmem. By raising vmalloc, you reduce the lowmem range.


**Strategy for common memory map (cmm)**  
Both system have their own memory map. A common memory map will serve as a synchronization tool to make sure one does not impede in the domain of another kernel. Some chunks are exclusive, while other are supposed to be shared (example: zones which are dma-ed by drivers). The case of shared memory makes not much sense actually, because it should not be regular RAM. It is the limitation of sparsemem that forces me to use it this way.

The common memory map model I propose has several heavy implications : every kernel must use the same memory model, with the same kernel chunk size.

Per kernel, memory chunk can have three states : shared to all 0, online to kernel x, offline to kernel x.

The common memory map should be stored in the fsw_page. For the sake of simplicity, we will use one byte for one chunk. By experiment, the chunk size can not be too low, I seems to recall it is 8Mb per chunk. This could potentially lead to an explosive number of total chunk. The size I choose is 16Mb, total number of chunk is 255, and the working range is 128-186. If chunk size would have been 8Mb, 512 bytes are necessary, which is equal to a 0x800 plage. This is too big. We will setup another page for the CMM, so we are not bound. The address in the beginning of fsw_page is used by boot params.

Each byte of the cmm has those simplified values :  
8 low bits indicates the owner of such memory :
> 0 indicates no owner (either free or no memory, see other flags)  
> x indicates the owner kernel (either claimable or not claimable)  
> 0xff indicates shared to all (necessarily not claimable)  

The next 8 bits are flags
> 8 (FSW_CMM_AVAILABLE): 1 = available : if chunk has owner, it is claimable. else, it is free (to claim), 0 = not available : if chunk has owner, it is part of kernelcore, else it is no memory altogether.

A more advanced treatment may be needed.
> 9 (FSW_CMM_RELEASE_ORDER): 1 = release order  
> 10 (FSW_CMM_RELEASE_RET): 0 = release failed, 1 = release succeed  

One may add a new field in the sys/devices/memory subsystem, to indicate the state of the memory (owned by who, claimable, etc...).


**cmm initialization by master**  
The master kernel will initialize the whole table : chunks have no owner and are not claimable. Then it will write at boot which chunks are his (the whole System RAM range), which are movable (set FSW_CMM_AVAILABLE to the re movable zones : we thus protect the kernelcore zone at least), and which are shared (change the owner, it seems this must be a hardcoded operation).

Setting up which zones are movable is actually tricky : some zones are noted as "removable" (like where the kernel is) but will fail eventually. As we have no way to make the distinction, let's rely only on this information. **This assume the removable information is definite and static**

The mem option should indicate the "maximum memory amount available", in case the common memory map has not been found (for the child kernel).

**loading a second kernel with cmm**
Requesting first to offline the continuous chunks of memory where the kernel is expected to live and be extracted : 0x9600X, 0x9700X and 0x9800X. Those are getting extracted, but should hold correctly within this range. This is the strict minimum the kernel is expecting, as this range is determined at compilation.

Next, as vmalloc has a fixed size of 728M, kernelcore 256M (16 chunks), we should understand that lowmem has at least 128M. Are the chunks given to kernel necessarily continuous? this is to test. Anyway, the master kernel should free enough chunks and set those chunks' owner to the instance to load, claimable (AVAILABLE, OWNER=2). now it should be free to load the zImage and initrd, then switch boot.

Next, the second kernel will load: this is a **critical point**. Here, a problem rises : a chunk of memory must be in System RAM so it can be added to sparsemem. I can fool it by using scanning the cmm, and use add_active_range to indicate what memory belongs to second kernel (OWNER=2 or 0xFF) instead. By doing so, the second kernel will have the memory master kernel has freed to him.

The problem is now "how to add the other ranges of memory but not use them". I think it may be possible, but i need to study how the memory gets offline (use register_new_memory in drivers/base/memory.c).


**cmm update**  
Making offline a memory range should change the state of cmm. This can be done either on switch (memory map scanning, then changes bits), or when the memory is manually set to online-offline. For the sake of putting code at the same place, the scanning solution seems better. But in dynamic change (next feature), a request of onlining or offlining memory must go through the inspection of fsw_page, and eventually a suspend request. Therefore it may be necessary to put code in places where online/offline is done.

Offline order :
> Check if memory is AVAILABLE. This is because memory allocated to another kernelcore is marked as not removable by another core, and should not be touched, therefore any NOT_AVAILABLE memory will never be freed.
>> If so, go through the normal operation, and if it succeeds, change the state of the chunk to NO_OWNER, AVAILABLE.

Online order :
> Check if memory is not already online  
> Check if memory is AVAILABLE.
>> If AVAILABLE, check if NO_OWNER
>>> If NO_OWNER, then it is free, so claim it and change OWNER to the current instance  
>>> If OWNER, then this is trouble. Set chunk as RELEASE_ORDER (as well as a common release order flag), initiate a suspend.
>>>> Release order: resume, offline any memory marked as RELEASE_ORDER (mark it as NO_OWNER, AVAILABLE, and RELEASE_RET, else set RELEASE_RET accordingly if it did not worked), and sleep back.  
>>>> On return to the first system, check which have RELEASE_ORDER and RELEASE_RET, and claim ownership of the segments who have been successfully released.  
>> Else, return error (potentially, you could check if it has an OWNER and set an error message)

Kill order :
> Just check which section belongs (OWNER) to the instance to kill, and force a claim.

**unregister memory**
To remove some memory from the sysfs interface, one can do unregister_memory_section(__nr_to_section(<number>)). This can be used to forbidden access to upper sections, ie the ones who where causing trouble. Good news. Very good news.

Removing the TUNA_RAMCONSOLE and DUCATI from the sysfs. They will appear in /proc/iomem, but not in the /sys/devices/system/memory and /d/memblock/.

Remainder : the sysfs depends on CONFIG_MEMORY_HOTPLUG_SPARSE, not CONFIG_SPARSEMEM. But the whole system will not work if SPARSEMEM only is activated : indeed, to load a second kernel, one must use MEMORY_HOTPLUG as well to unload memory. FLATMEM should be usable as always and should not require cmm.

### 30/08
**Implementation of cmm initialization**  
There is an issue about this. At initialization, there is no way to know what data is removable : after all, the system has not finished to initialize, how are you supposed to know? We at least know that ZONE_MOVABLE are relocatable. We set those to available. It is perfect because in our current configuration, 0x8000X to 0x9000X is not movable and exclusive to the first kernel.

A second problem is the status of shared memory. This memory should be platform specific, and so the modifications to add are in board-tuna.c, after CMM has been initialized. This memory will be marked as shared. The table is correctly initialized from master instance pov.


**Memory initialization via cmm**  
A problem we may encounter is that the memory initialization happens extremely early. This supposes that we read the cmm content as well as some fsw flags to determine if we are loading a second kernel.

By default, sparsemem sets to online every memory it founds at boot. The process goes by this: sparse_memory_present_with_active_regions() will initialize mem_map with a SECTION_MARKED_PRESENT flag. It is later, in memory_dev_init, that the sections are added according to present_section_nr() with the add_memory_section() command. This also means that the shared memory MUST be added in active ranges.

The strategy would be the following : on booting, in bootmem_init (where we populate sparsemem), we will add the intersection of allowed (or found) memblock with the CMM.

I have an issue in reading the content of FSW_BASE. MMU is enabled already, but ioremap does not work so early. Thing is, ioremap is likely to use the regular mem_map, while during initialization it is another mem_map that is used (bootmem_map). And this map remapped only lowmem memory. (Also, raw_readl will not work).

It would have been easy if ac00X belonged to lowmem (in case a translation would have been sufficient). But it is probably incompatible with loading several instances.

The problem of the FSW_BASE not being in an easily accessible page is nerve wrecking.  
* One possibility would be to temporarily shutoff the MMU, copy the bootmem_map from FSW_BASE to a kernel accessible address (in LOWMEM), then resume the MMU. Done this before. But this is highly NON portable, because I will temper in platform independent code in arch/arm/mm/init.c. This is B-A-D.  
* Another one is to add entries into the MMU table so this address is accessible. However, I don't know if the mmu table is open at this stage.  
* Maybe the most convenient solution would be to copy a copy of the CMM somewhere in the memory (like 80001000), so it can be directly accessed in bootmem without altering the table. This has a drawback in that fast_switch must copy one more data, and it may be potentially difficult to find such location. The address of copy is also arbitrary and fixed, and the kernel must be compiled with the proper value. Later, it may be possible to specify a location via a commandline.

A alternative method would be to alter the memblocks, and register new memory zone later according to the CMM. However, I still need to know how to access to the FSW_BASE.

Reading : bootmem allocator http://www.thehackademy.net/madchat/ebooks/Mem_virtuelle/linux-mm/bootmem.html

Reading : looking at mmu table http://lists.infradead.org/pipermail/linux-arm/2010-March/000147.html

**copy CMM out of FSW ?**  
The process is like this :
> Boot  
> Reserve the range we are expecting to read the CMM (which should be in the lowmem zone)  
> In bootmem_init, read if the table is here  
>> If yes, compare and add memory range accordingly  
>> Else, just free this location (the reserved one, it is ok since we are not expecting to copy) and let the regular operation do its job of adding ranges  
> Later, if it has correctly booted, synchronize the common table at 0xAC00x

The range where CMM is copied should be standard. Sadly, we are intervening in platform independent code, so it must be a constant (now it is FSW_CMM_COPY_OFFSET). Oh well...

On this location, a reboot will not rewrite it, so it is safe.

In the beginning, I wanted to copy the whole table with the whole owners. Then the second instance was supposed to know its instance number, then check the table, and compare. Now, since I have no access to FSW_BASE, I will rely on a slightly modified table, where only the sections to load appears. The values are still the same, ie ONWER=2, 0 or shared.

Implemented this, but there is a bug : Can't miss any range contained in memblocks. What a fail... but the framework for CMM out of FSW is here at least.

Another solution will be to alter memblock. Let's see. With memblock_remove and all, it should be possible to do things correctly. And it's a jackpot, I managed to work with the previous solution by removing the memblocks range which had no active_range with memblock_remove.

So to simplify the implementation, I should remove in a very brutal manner, any memblock that is not in a section. To be specific, any section without owner will be cut down and memblock_removed. The policy being if any section is not in the cmm, it must be removed. The other problem, however, is that we can not randomly remove ranges that are not in memory. But it seems the case only if the address we want to remove is below PHYS_OFFSET (the minimum memory?). The remove from the CMM table works well now.

Notice: memblock_is_memory check whether the given address is in a memblock, and memblock_is_region_memory checks if the given address + size belongs to a memory.

In the mean time, I tried to add the section I removed, and although it appears in the sysfs, offline, it can not be changed again to online. See drivers/base/memory.c, memory_block_change_state(), further it is walk_system_ram_range(). I is because the range we want to add is not defined as "System RAM", therefore it refuses to add it. This is something to do at second kernel initialization.

**Supplementary details**  
A booting kernel may want to claim any free page ? Or only the sections the master kernel told it?

Notice bug : The section 172 is still not readable because of the AC00X page (it is exactly the same problem as RAMCONSOLE). If an access to the mem_map is done, kernel will try to read pages in the beginning of the section (which is blocked because of me **(is it reserved or removed that makes a problem?)**). This might be one of the reason why I have sudden crashes sometimes.

I try to move FSW_BASE just after the RAMCONSOLE at 0xA020X.

It is not all, because asking if the AC00X (section 172) range is removable makes an error. I think it may be because of the hole at the end (the memory is be 0xAC00X - 0xAC9FX, higher and there should be some other artefacts or other kind of memory).

Bug happened again, and this is not very good... It happens even if no memory has been taken out. This might be due to SPARSEMEM not being stable on the GNex.

Happened again, and saved a last_kmsg report. It is a bug in mm/page_alloc.c, where page_zone is different between start zone and end zone. It may be due to CONFIG_HOLES_IN_ZONE not properly configured. Maybe I should add this to the config?

TODO tomorrow:  
* Update CMM when a section is offline
* Interface to copy a CMM for an instance
* Add the memory which is supposed to be RAM inside System RAM (according to the real CMM)
* CONFIG_HOLES_IN_ZONE

Notice: ARCH_POPULATES_NODE_MAP does not depend on MEMORY_HOTPLUG_SPARSE, but allows the use of movablecore/kernelcore.

### 31/08
**when to copy the CMM**  
The best course of action would be to copy the CMM on switch-boot. Therefore, fsw needs the address of where to copy. The provided debugfs interface needs only the address to copy. The instance number will be provided by the switch command, and on switch time the CMM table will be sparsed for correct values.

Notice that this work is redundant, and that the best course of action would be to access to the fsw memory directly instead of copying to lowmem.

**ioremap in System RAM**  
A future problem : during the boot-switch pre-check, I was relying on ioremap to access the physical ram (which was outside of the proper ram anyway). Since now the whole ram belongs to master instance, ioremap can not be used anymore : ioremap makes a check if the target addr has a valid pfn (apparently it causes problems on ARMv6+). A regular map must be done. However, the memory here is supposed to be offline... It is getting complicated.

ioremap works only if zone is removed from the memblocks, at least that is how I access to FSW_BASE and FSW_CMM even if they are in System RAM. Reserving them has its use though, so we know kernel won't put things inside when we operate.

A bug happened when I tried to investigate whether there was a need to remove FSW_BASE instead of reserving. I understand that if we do not remove, then it won't be mapped via ioremap. A bug happened once, and now there is indeed no absolute need to remove it. At term, we may want the two FSW_BASE to be inside System RAM (where we are sure there is plenty of available memory), without removing it. Maybe we should do a transition to mmap.

But for now, a ugly workaround is to memblock_remove, ioremap, memblock_add : this supposes that the block was originally inside a memblock, of course, and this is something to check thoroughly to avoid memblocks of unexisting memory. However, it worked only in the beginning at boot. Did not work for a offlined section. The block was correctly removed, but ioremap failed.

After more investigation, it is because I was working on a corner case: **I removed a section of a page**, from 0xff to 0x1000. This was enough to consider the pfn as valid, hence making ioremap fail. When removing memblock, we need to remove the entire page (or at least its entry) so apply a PAGE_MASK.

This workaround can actually be useful in covering both cases. However, it is unsure whether newly added memblock are covered by mem_maps (unlikely?).

This problem raises the question of whether what memory should an instance consider to be System RAM. In the Flatmem model, each instance could only see its System RAM, and anything outside was accessible via ioremap. With the sparsemem model, since we want to exchange memory between instances, they are likely to see any memory exposed to them (by the CMM).

Notice : ioremap on ram is BAD
* http://lists.infradead.org/pipermail/linux-arm-kernel/2010-October/028490.html
* http://lists.infradead.org/pipermail/linux-arm-kernel/2010-October/028516.html
* http://lists.infradead.org/pipermail/linux-arm-kernel/2010-October/028760.html
In a nutshell, it is because when a memory has many different mappings with many attributes, behaviour is undetermined.

And at the end of the day, we propose to do exactly like what ram console is doing : remove at boot, so it can be ioremapped. We will keep this behaviour from now on. The issue remains on copying image though.

**testing offlining memory**  
Some bugs happen after I offline memory, I think this has to do with CONFIG_HOLES_IN_ZONE. Damn : looks like almost no architectures have this attribute. What does that mean?

Took a look at the failing page, it gives me this : pfn fe06b000 0 pfn a03ff 2, in the section around 168 (a8). Curiously as well, why is there action from a pfn around a03ff? I never asked this one.

It is totally strange for a pfn to have this value. It is one particular page that makes things fail. I wonder what it is...

For now, we just ignore the check. Our map indeed can have hole in zones.

**testing loading second kernel**  
When offlining memory, I set CMM automatically so kernel 2 is the owner of those ranges.
Also added echo 0x96001000 > /d/fsw/cmm to superload.sh script.

It runs. And I am relieved...

Offlined zones are 0x9600X to 0xAB00X with 2 zones in the middle. This makes the ram to be pretty scarce (304Mb only), making the system very slow to run. I would propose to start from 0x9000X instead! Lowmem has the usual 128M, so the .config setup of mem= and vmalloc= does not matter much.
To make the matter worse, quite a lot of memory is reserved at boot, due to the obligation of master instance to reserve which zones are potentially rewritten : **we could actually free them latter**.

Like usual, it works only when **both are the same kernel exactly**. No idea why.

When trying superswitch, I remember that I made a check on the jump instruction not to go below PHYS_OFFSET. I should change that and make no check altogether... for the time being, it is not possible to jump back so this kernel is nigh.

Instead this time, we will try with starting the kernel at 0x9000X (section 144). The best result would be to get 22 sections to achieve the same result as before, and at least 8 continuous sections from the memory start. Succeeded in opening 22 sections from 0x9000X. After the last section was offlined in master, many services were killed.

This, however, **requires to modify the atags which register the location of initrd**. For now, I specify them in the boot args so there is no need to modify the atag file. However, the time will
come where this is necessary.

Tried, and failed, there is an exception, I have no idea from where. It is the kind of violation access to somewhere... don't ask.
However, it works good for 0x9600X and the transfer has no problems.

I located the issue, it is in mem= bootarg. <starting addr> + <mem size> should not go over the real maximum ram at 0xc000X. Therefore, 1G@0x8000X is ok, 672M@0x9600 is ok, but not 1G@0x9600X.
Trying 768M@0x9000X. vmalloc= does not have any influence.

It worked.


**Next steps**  
> Registering missing memory on boot for second kernel : so we can see a proper view of the whole mem (and at the same time remove the b3 range? Maybe not a good idea since those sections are supposed to be online on boot, so the driver does not crash)  
> CMM update strategy : it would be good to rethink of a proper strategy for memory allocation on switch boot. Maybe make a semi-automated process  
>> This include ? for example, give start address (0x9000X), total memory (352M), minimal contiguous kernel memory (128M), and the address of the RST  
>> The system will be in charge of offlining memory according to what is available and update flags of the cmm. If all sections required sections have been freed, we proceed to build the RST. else we restore (set online) the offlined memory.  
>> The CMM will be copied to RST only when switch-boot is ongoing OR (the strategy we adopt) when the memory has been freed correctly (this will of course require that the target address for RST is free).  
>> About the flags, when offlining the memory, we can set the flags of the CMM to empty (no owner) (and set back the flags on onlining). The booting kernel will thus modify which sections belongs to him (and set the proper flags).  
> load_image and offlining : We may want to refuse doing load_image if section has not been offlined.  
> change the mmap mapping from ioremap  
> set fsw_page as a IORESOURCE_MEM, via request resource.  

**registering absent memory to System RAM**  
arch/arm/kernel/setup.c see request_standard_resources()  
Question is now, when to add those resources? At least when we have access to the CMM. The first call to request_standard_resources() happens very quickly, just after paging_init() (bootmem_init()). This could means that we have no access to the full CMM, except if we make a full copy of it (hence the RST).

There is a caveat: we want to restore the System RAM as it was meant to be. We, however, need to know what was originally the state of the memblocks. Since we removed some memblocks, this state is lost forever.

This raises a question of whether we should add the whole memory instead (leave the memblock untouched), but offline the section afterwards instead.

The solution to remove memblocks first makes sure no memory corruption happens, so we may want to keep it.

Another solution to this, is that before erasing the memblock map, we make a copy of the regions, so we can retrieve it later to define the System RAM. This is not that trivial.

For now, we may not want to worry about holes in section. Let's just add whatever section we have, and worry about the holes later (or forbid any section with hole removal).

A third solution, is to mark in the rst what memory belongs to the System RAM.

A fourth solution is to find a place where we can remove memory safely, after the sections have been added (done by sparse_init) and after that System RAM has been populated.

Trying this tomorrow...


**adding a resource**  
The function to do this, however, is simple : mm/memory_hotplug.c add_memory(). This is the highest level in adding memory and should be used.

This is already the biggest problem : if we can not hotadd memory, we can't do anything.

It has issues though, adding new memory works but crash soon after the onlining.
* Unable to handle kernel paging request at virtual address d6c00020
May be due to the absence of memblock.

Adding the memblock before doing add_memory, makes the block appear in System RAM, in /d/memblock, but not in /sys/devices/system/memory.
Adding the memblock after does not help.

It is a problem in the slab allocator ? ... I see in documentation http://www.kernel.org/doc/Documentation/memory-hotplug.txt that newly added memory is added as ZONE_NORMAL. We are expecting to add as ZONE_MOVABLE, so I think there is a lot of work in understanding what effect it has.


**Chen is putting pressure**  
Wants to switch back and forth with release orders. This would be possible, but tricky.

Master wants to reclaim an amount of memory : he ask random systems until he gets his quota.  
Child wants to claim memory : only ask the master.


In the future, when two cores may have access to the same memory, we will have cache coherency problem. So far, it was not a problem since only one system was doing work in those zones.

Notice : learnt what it meant to be ISO C90. Declarations must be put after a curly brace.

Notice : Maybe some reorganization is necessary. Instead of copying to an arbitrary address, we may provide one base address and mayke every copy as an offset of the base address.


### 01/09
**Why full mapping at boot**  
Instead of having the memblock removed at boot, we will show them all and remove the sections instead. The assumption is that the boot operations are doing the job in bootmem, so everything is in low mem. We need to offline the memory early so the ZONE_MOVABLE doesn't get touched.

Another assumption is that the memory allocated for the instance by master must have at least enough place to hold kernelcore (or at least, for the lowmem range).

free_area_init_nodes() is seemingly necessary to build the pg_dat, count the ZONE_MOVABLE pfns, etc... Therefore this seems a sensible operation to let the memblocks initialized instead of removing them. In theory, initializing the memblocks should not touch the content of the memory it addresses.


**memblock_reserve on boot**  
A temporary solution would be to reserve the memory of unavailable sections, so we can offline them later. The thing to do indeed is to ensure the kernel don't use those locations. By reserving those locations, we see that the available memory from /proc/meminfo does not take into account those blocks.

Those reserved blocks are seen as online, but they are not offline-able though.

Notice : Doing so gives the only role to the CMM copy of "indicating which memory child instance should not use", hence it's name: reserved section table. The reservation however is zealous, and it will reserve even ranges that are not memory. It should be OK to remove those ranges from the reserved pool later. Those sections are "to not use", while not being kernelcore memory. Therefore, they have no reason to be reserved. But a better strategy would be to strictly reserve the sections.  
But the issue remains : once reserved, I can't seem to put the sections off. They can not be offlined even if I memblock_free them after memblock_reserve.


**understanding memory online-offline**  
Doing offline page starts from remove_memory() at mm/memory_hotplug.c  
With some investigation, a pageblock is offlined if any page within the pageblock (memblock) is "isolated" : this means no references (?) to it, or free (belonging to PageBuddy).  
In our case, a classic removable section belongs to Buddy (and so can be offlined), while a memblock_free-ed page belongs to nothing. This probably means that memblock_free does not make available those ranges. There is a missing link.

This means somewhere after the boot, between the moment I do memblock_reserve and memblock_free, a function allocates those zones to Buddy (tested by memblock_reserve then memblock_free immediately, whole memory is available). What's more, a offlined section reduces the memory, and re-onlined section adds memory, indicating that the Page buddy is modified.
This also means that memblock_free does nothing.

This induces the need to locate how offline memory gets added to Buddy when set to online. Or reserve those pages later. Or make them free after they have been reinstated (like free_area in arch/arm/mm/init.c, or online_page() in memory_hotplug.c).

**onlining reserved pages**
Yes! Managed to bring back to life a reserved region, that was tedious. Using online_pages(pfn, page per section), which will initialize the pages by setting them to free (and probably add them into Buddy).

This method has two implications :
* we can only online offline entire sections, we can not compromise with holes or partial sections.
* online_pages does not work on "out of range" regions, therefore we have to change the format of the RST by indicating "what is to reserve" (no more reservation of out-of-range memory).

Using online_pages, we have a sysfs indicating that those pages are online and used. We can offline them manually after. There is even no need to memblock_free them, except for the cosmetic. Using memblock_reserve to do the job is just a nice and easy way to protect the memory.

The only concern I have is that during the onlining process, **memory inside the reserved section gets changed (?)**...

However, the state of sysfs is not up to date. sysfs happens to be already initialized though (memory_dev_init() in drivers/base/memory.c). This is because it adds sections directly thinking they are ONLINE (which is partially true). The changing status is only a sysfs internal variable.

    struct memory_block *mem = container_of(dev, struct memory_block, sysdev);

**synchronize state of memory_block and section**  
It is useful to sync those two, so we don't make any errors. It is possible to get the memory block if we know the memory_section.

    struct memory_block *mem = find_memory_block(__nr_to_section(i));

And modify mem->state


**online and offline sections by code**  
A summary:
* online by online_pages()
* offline by remove_memory()
* change the state of the block with mem->state  
If offline fails, check __test_page_isolated_in_pageblock in mm/page_isolation.c, this is where linux checks whether the pageblock content is free/isolated or not.

This method should be used to make offline-ing section automated.


**memory state and CMM update: synchronized?**  
For some reason, it may be desirable to set offline a memory separately from the CMM.

Also, the code to remove memory (remove_memory) is in a arch independent code (in mm/), so I am not too keen on modifying it. I propose to expose a specific interface to free memory in /d/fsw, which will have control over the CMM. Although, now, I have an issue regarding the modification of regular sysfs interface : indeed, if we want to remove a ram, there is a need to ask first whether CMM is ok with it.

The implications is that we need to add control code inside remove_pages and online_pages. Or change the code inside the memory sysfs subsystem.


**"freeboot" memory sysfs interface**  
Remainder : start address of free mem, kernelcore (contigous), total, start address of rst
Therefore, the table will be built on command, and not on suspend.

An issue I stumble upon is the fact that I offline a section, but I can not online it again
* section number 145 page number 0 not reserved, was it already online?
Therefore, my operation is not strictly the same as the interface.

After several tests, we have :

    remove_memory(0xa2000000, 0x1000000) working
    remove_memory(0xa2000000, PAGE_SIZE*PAGES_PER_SECTION) working
    remove_memory(i << PA_SECTION_SHIFT, PAGE_SIZE*PAGES_PER_SECTION) working

It is because of the type of i, which MUST be unsigned int or unsigned long. I made the mistake to set it as a int, and therefore a sign error was present when doing
So the operation is correct. Just beware that remove_memory return 0 on no error.

Returning to memblock_reserve on boot, there is a problem : the section 0xa000X can not be offlined. The body of this section is in memory. Therefore child instance should map this section natively. To avoid the child mapping this section, we could ask the child to reserve it then offline it. This section however, is not offlinable, because the head is out of the memory. A fix to this would be to disable the full section for all kernels, or to move the RAMCONSOLE out of the head. A better fix would be to change SPARSEMEM.

TODO : beware of mutexes, there is a lot to do in protecting the data structures

TODO tomorrow :
* finishes the interface to load memory automatically, and boot second kernel with it
* try manual offline and online, going back and forth
* implements the auto jumpback  
I guess, tuesday should be ok


### 02/09
**freeing versatile memory in freeboot**  
The part to free contiguous memory is mostly ready.
Sometimes, there will be a hangup: this means the kernel is trying to free some memory, and is waiting for it. This may also mean that there is just not enough memory if it timeouts. Reduced the time to wait to 2 seconds.

--

Made quite a lot of changes... So now, /d/fsw/free_boot will free the sections, change the state of the CMM, and copy a RST if the target section has been correctly offlined.

Resolved quite a few bugs in the process.

The booting kernel will in turn check the RST table and so on... this part has not been modified.

There is however another issue, it is the ac00X block, which is partial.

The problem is the following : 0xac00X-0xaca0X is the working range of this section. Since it belongs to the master instance, it is notified as CLAIM.  
When the child instance run, it will reserve this section. It will also take it and make it online then offline. Since we tell them to offline the whole section, it wont be too happy and fail. Therefore, I think this zone is no good. Now, question is: does it happen for every "memory range end"?  
TODO Anyway, this is one more section to remove and to reinstate later.

I thought I could just block AC00X range, but the AD00X range popped up in memblock/memory. I wonder why? I think there might be one more driver that is doing something in this range?  
On another hand, when I blocked 0xAD00X range, AC00X and AD00X were both correctly removed. That is quite surprising... this is a work around, for now.  
TODO investigate this issue

Hopefully, this will make both kernel finally having excluding memory.

And it works okay, the kernel jumps correctly. (kernel1 = 0x8000X-0x9000X + 0xA500X-0xAC00X and kernel2 = 0x9000X-0xA500X).

I think I should raise the timeout for offlining pages, sometimes it fails.

Onlining and offlining manually memory works. Good!


**proofcheck in online and offline pages (CMM update)**  
To avoid onlining pages that do not belong to current instance first.  
The highest non-static function is online_pages() in mm/memory_hotplug.c. It is not very good, because mm/memory_hotplug.c is a arch independant code, and we want to make checks.  
We want to make a online attempt fail. Let's modify the change  

When offlining pages, the work is only to update CMM flags, therefore it is easier. This can be done easily in remove_memory() at mm/memory_hotplug.c, where it is easy to add something.

Online->Offline request cases (remember that offlining migrates the page content elsewhere):  
* owned, not available : OK, no change
* owned, available : OK, and if succeed, section is set as no owner, available
* shared, not available : NO, no change
* shared, available : NO, no change  
(below are impossible cases, since online sections are owned by me or shared)
* not owner, not available : NO, no change
* not owner, available : NO, no change
* no owner, not available : NO, no change
* no owner, available : NO, no change

Offline->Online request cases:  
* owned, not available : OK, no tag change
* owned, available : OK, (strange case) no tag change
* shared, not available : NO, no change
* shared, available : NO, no change (should never have to online shared sections, actually)
* not owner, not available : NO, no change
* not owner, available : REQUEST (NO)
* no owner, not available : NO, no change
* no owner, available : OK, if succeed, change the owner tag

Implemented, it is a simple check and it works well.


**strategy of freeing/claiming memory from other instances**  
An example:
* An instance lacks memory. It wants to reclaim 2 sections. It writes a section reclaim request, says it is instance 2 that is claiming 3 sections (claim_count), that the sections must be higher than its base (0x9000), then goes to sleep.
* The first instance will wakeup, understand that this is a reclaim request, and attempt to free some memory in its own pool of ZONE_MOVABLE where we make sure that the sections we attempt to offline are
>> 1/ higher than instance 2 base address  
>> 2/ belongs to AVAILABLE (which is the intersect of every ZONE_MOVABLE, and so we are sure that it belongs to instance 2 ZONE_MOVABLE)  
>> 3/ belongs to the current instance  
* To know whether we have enough memory, we can attempt a remove memory, one at a time, until all have been tried. For each successful remove, we reduce the claim_count by one, and we mark the section as free, and "RELEASED". We check claim_count each time a section is successfully removed, or once we have no more sections to check.
* If all zones have been tried, but claim_count is still positive, we attempt to jump to the next instance that is not instance 2. It is easy to check, by looking at the fsw_page, if there are tags.
* Once claim_count is zero, we jump back to instance 2
* If all instances have been tried (according to MAX_INSTANCE), then we failed, and we jump back to instance 2.
* On resume of instance 2, we check all the RELEASED tags, and claim ownership of what we have. If we reached our quota, that's good.

Selection of the target will follow a round. We start from the first


My partition bugged again
* http://productforums.google.com/forum/#!msg/mobile/dljRq2w854Y/k8WuL0TaIIgJ
This time, both systems were corrupted.  
The userdata was just unusable because a folder property could not be read.


### 03/09
**strategy of freeing/claiming memory from other instances**  
Implementation continues.

Ideally, before jumping to one instance to check if it has memory to free, we could check if it has movable memory.

To allow a loop in suspend, I moved the fsw operations just after IRQ are disabled. This would cause a bug coming from ioremap (illegal context to call).

Then, to make a direct jump, it suffices to use the existing suspend framework (no need of a supplementary mode I guess), to reset the target correctly, and set allow_core, then switch
```c
suspend devices
cpu off
**fsw pre-operations**
irq off
syscore suspend
suspend
syscore resume
irq on
**fsw post-operations** and potentially jump back to before
cpu on
etc...
```

We can also, in the post operations block, offline a memory. It works out of the box, linux is awesome.

Now, the problem is we SHOULD use the suspend mode instead, but add an additional tag. This is because suspend mode and claim mode have a common denominator at the lowest level (assembly code), therefore there is no need to rewrite code for the claim mode case.

**Rethink of the strategy ?**  
We can use suspend_mode as an entry point for the next jump.
```c
> setup claim count, source, first target instance is master. mode is claim mode for now
...
> claim_mode label, which handles target selection.
>> Select source if all instances have been depleted, or if claim_count is null
>> select the next valid instance if claim_count is not null
> suspend_mode will check the validity of the next target, and set allow_core
> ... suspend-switch ...
> ... resume ...
> if claim source is not written, we skip: we are in a regular suspend-switch operation
> if claim source is written, and we are in the target instance (!= source_instance) : << this case happens for every present target that is not source instance
>> if the claim count is not null, free memory, reduce claim count, stop once claim count is null
>> **jump back to claim_mode**
> if claim source is written, and we are in the source instance :
>> check whether we managed to get the memory we expected, and claim any section that was freed
>> reset correctly claim source, claim count, print any messages, etc...
> if claim source is written, but an error occured (meaning we could not jump to target)
>> if target is not source, then we jump to the next valid target ie jump to claim_mode
>> if target is source, we either retry to jump back, or abort and stay in the current instance
```

current instance tag should never change during the full operation... or so I thought. current instance must strictly be the instance undergoing suspend, because it is this value that determine where to save the sar ram.

Working good!


**optimizations**  
```c
- need to rework freeboot *
> When we freeboot, we savagely remove memory. But we should first check that this memory belongs to us!
- reserve at boot for the child instances
> we have now the cmm, we know we are a child instance early so we don't need to reserve memory
- proper mappings * (only the big copy is left)
> don't use ioremap
> change the rawreads to classic memory access (because now we are accessing the whole memory as memory)
- make common code become functions * (partial)
- clean up
- try reservation of fsw page and cmm during common fsw_init, not in tuna board
- find a solution for the timeout in page removal = let is be...
- resolve the mystery around the bug of pagealloc (HOLES_IN_ZONE)
- difference between kernels
```

**future work**  
```c
- expect to run many oses on many cores
> there will be an issue with the "main core" not being cpu0
> there will also be other issues in shared devices and things
- periodic time sharing between two oses
> 5 sec in os1, 1 sec in os2, 5 sec in os1, etc...
```

**focus on paper**  
This is the absolute requirement, 70% of the time on paper, 30% of time on other work, no compromise.


### 04/09
F*ck, lost what I have written.

Check unionfs, it is an interesting finding in supporting filesystems.

An idea is to save the framebuffer of instance A, before switching suspend to instance B, so we can resume it just before switching suspend to instance A again.


**About Cells**  
In a nutshell, a virtualization mechanism geared toward smartphones. The single unit of execution is not a whole OS stack (namely Android + Linux), but a VP (virtual phone), which consists of a Android userstack. Many VP run at the same time over a Linux kernel, and share (or not) the device hardware.

**VP isolation**  
The isolation of VP between each others rely on the namespace feature of Linux. It is a set of features which can make a set of process run in their own world. namespace including processes (so they don't affect each other), user id (so a root user of a VP is different from the root user of another VP), mount namespace (so a mounted device is not accessible by another VP), and removes the right to add nodes and devices (so remounting is not allowed) and data are protected. Filesystems are thus protected.

**device virtualization**  
Cells introduce device namespace for device sharing. Each VP has a device namespace for each device, used for device interaction. The device namespace knows if the VP is foreground or background.

First method, like the FB, is using a wrapper of the device driver and according to the set of permissions, the wrapper returns a access denied to the VP or control of the device driver. It multiplexes access to a common resource.

Second method is to modify the device "subsystem" and make it aware of the device namespace, ie send the command to the correct device namespace. Case for input.

Third case is to modify the device driver, it is about the same method.

**root namespace and userlevel device**  
The fourth case concerns user level drivers like wpa_supplicant, and introduces proxies to the services. It works like this:

Among all the VP, there is one namespace (which is not strictly a VP), the root namespace. It is the first to run, and initialize among other things the wifi, radio, etc... like a central server for hardware access. I guess they can not use the previous method since initialization is done in userspace level. Then CellD is also responsible for starting or switching other VP. VP to start some device confguration, sends command to the root namespace. Communication between VP and root namespace uses IPC sockets.

**hardware access**  
foreground and background VP is the key difference between all the VP. The foreground VP has access to all hardware. For some device, foreground VP has exclusive access, or shared access, or no access. Background VP has shared access if it is in its set of permission, or no access.

**scalability**  
Clever use of KSM which merge pages of the same memory. It avoids copy of the same page, and can be used only on the same kernel of course.

Clever use as well of aufs, a union file system, were the visible fs is actually the superposition of a writable layer over a read-only layer.

**Framebuffer**  
In the case of the framebuffer, only the foreground VP has access directly to the screen. The background VP still run, but has no access to screen. Cells replace the framebuffer driver with their own driver. All VP renders to their own buffer in system RAM, but only the foreground can render on screen.

In the case of Gpu, there is no need to make distinction : since all process are already protected by their namespace in their own context, they only need to send commands to the GPU and to make it render in their own backing buffer. The framebuffer driver will mmap the correct buffer (the foreground VP one) to it.

**Powermanagement**  
not much to say, background VP can't initiate suspend, and the fb earlysuspend is blocked (they see the screen as black). The foreground VP has all rights (it still depends on the wakelock though).

About wakelocks, there has been a lot of thought but did not read too much into detail.

**Telephony**  
They add a double layer between two interfaces, the RilD (radio interface layer daemon), which dynamically link to the vendor RIL. RildD used to connect with RIL. Now, instead, they use a user-level proxy which will communicated with CellRIL, which will in turn communicated with CellD. CellD (of the rootnamespace) is the only one making links with vendor Ril.

A set of privilege (who can do what) is implemented (like only foreground can make calls). They describes a set of rules, but that was not very interesting

For their "multi number" functionality, they go through a voIP server which will redirect the call to our unique sim card. And they use a trick of adding a number ad the end of the caller ID, so only the receiving IP get the call.

**network namespace**  
This is where things gets interesting : the root namespace has control over the hardware (wifi, radio). For each VP who wants an access to the net, root namespace exposes a ethernet pair from the root namespace to the VP. Using NAT, redirection is done automatically. That is sly! Each VP has its own "virtualized network resource". However, configuration is done orignally via userspace (the VP). Therefore a proxy replaces the wpa_supplicant, and sends command to rotnamespace for initiating the connection.


### 05/09
**Refining the free mem strategy / mem_request**  
In the case of freeing AVAILABLE memory from other instances, I thought I could just free random available memory. But in the case of doing a freeboot, we want to explicitly free specific AVAILABLE sections. This requires to tell what instance we want to free. Doing this requires to mark which instances we are expecting, and switch to claim.

Resolving first this case... I think it is better to ensure that the memory belongs FIRST to the owner, before freeing the contiguous space. If the current instance has not enough memory, then it should first request memory from other instances.

This means that we need a way to request specific sections instead of "Random ones".

That aside, mem_request is taking mem from other instances, instead of taking from an available pool. We may want to claim what is claimable beforehand (among the pool of sections without owner), even if no arguments have been provided.

Corrected a bug in onlineing page without changing the state of the memory

Change the strategy for mem_request :
* if a second argument <start addr> is provided after <section_count>, then we want the <section_count> segments directly following start address to free and claim back.
To do this, we can add one more tag, called "CMM_REQUESTED". Only the sections marked REQUESTED will be taken into account for checking if they can be freed. At the same time, we will get any claimable memory without owner as well.
* In the case if no arguments have been provided, then any AVAILABLE sections will be marked as REQUESTED.

So this problem is more or less resolved. We can request memory from a specific location, or at random.

**ioremap**  
http://www.makelinux.net/ldd3/chp-15-sect-1
Took a look at pfn_to_page.

Yes, if the location belongs to a section (previously mapped), it becomes possible to do:

    page = pfn_to_page(pfn)
    addr = kmap(page)
    ...
    kunmap(page)

So we can map anything, as long as we are in kernel. I guess early kmap is out of question。

However, kmap works for small pages (there is a limited number of such pages). For loading the image, this would require more pages. Especially more contiguous pages.

**gsscript**  
allows to launch scripts. Good thing if I want to execute in one touch!

**jelly bean**  
About running jellybean and ics at the same time, there is a big big issue : incompatibility between the bootloaders and radio. maybe we can do a trick and just load a halfkernel ?

This would require first to solve the problem of different kernels not switching (although booting)...

### 06/09
**Discussed with David**  
... and the interesting project they have on running Android apps on x86.

**The problem of different kernels**  
> Extract from 26/07
Adding a pr_info line in arch/arm/kernel/process.c in machine_power_off() will prevent the system from resuming completely. Then the MPU watchdog will reset. I have no idea why.
After some investigation, the location of the bug is during the pm_restore_console().
If modification is made on son kernel only, it can not jump safely to master kernel as well.
Looks like both kernel need the same modifications to run correctly (at least, the arch/arm/kernel/process.c file). This is quite strange and will prevent two different kernel from running together.

Now, the problem happens in cpu_begin. I think there might be something to do with rewritten sections... ? Or another theory I have is the saving of SAR RAM. Among them one address is wrong.

After investigation, the cpu spin at one time

    while(gic_dist_disabled()) in arch/arm/mach_omap2/omap_smp.c

The problem seems that the cpu1 never wakes up and so cpu0 waits indefinitely.

cpu1 is supposed to die in platform_cpu_die and restart from there... NO! Seems like it is supposed to

So why, when the kernel is not the same, is cpu1 not moving?

Question is
* does it jump to the resuming kernel ? no
* does it jump to the point before restore? no

```
mrc p15, 0, r0, c0, c0, 5 @ Get cpuID
ands r0, r0, #0x0f @ Continue boot if CPU0
.equ memvale, 0x80001020 (or 0xc0001020 if mmu)
ldr r8, [r8]
movne r0, #1
movne r0, #0
strne p0, [r8]
```
Shows the last core going through.

That is very, very strange. I think something goes wrong in the initial kernel.

CPU1 does not seems to jump to the proper address in kernel 0. CPU1 is supposed to be in a wfi state, and come back to life when CPU0 calls it.

Just adding a few lines in sleep44xx.S does the same effect.
I know at least that
* the CPU1 gets woken up,
* jumps to omap4_cpu_resume,
* pass the flag check
* fails to jump to suspend_switch: it seems the mode is 0, which is strange because it is supposed to be 2 (core0 indicates the proper value).
* fails to reach the check for the jump address (ie I don't know by what magic it manages to jump to switch-jump)

It works when the kernel is the same, because CPU1 will completely skip the jumpback to first kernel. However, it will complete the classic path, and restore the MMU. Once MMU is restored, pc goes back to c0XX, therefore it will magically jump back to the first kernel. If kernel are the same, the address is valid. If the kernel is not the same, then the jump occurs somewhere in the middle, and so it will not work.

Moreover, the reset of the switch suspend occured after the core1 came back to life. But I changed that, and moved the state reset before. That was the reason why the mode got changed. This is seriously two bugs that cancelled each other, what a finding.

We are finally back to square one... different kernel will jump, but hang.  
In one case, the first kernel gets to the framebuffer, but the touchscreen is dead.  
In another case, we just can't get to  
This bug is much more difficult to track, because depending on where the supplementary code is added, it hangs at a random location. This is very similar to the time where I had corruption on resume, due to things being rewritten be second kernel.

* When adding a line in arch/arm/kernel/process.c, c00577f8. It will bug.
* When adding a line in sleep44xx.S, c006f340, has no effect on the crash. Therefore we may want to go back and locate this way ??
* When adding a line in arch/arm/mach-omap2/pm.c, c0067ccx, no effect.
* When adding a line in arch/arm/mm/vmregion.c, c005fdc0, no direct effect, although it will crash later.


From my observation, when I add the code segment at the end of omap4_cpu_resume, it does not trigger an error. When it is removed, it can trigger an error, but not systematically.  
And now, almost no error is triggered although both kernel have lot of differences... This may mean that only a specific difference will trigger the bug. This has something to do with PAGEs and MASKs, maybe?  

Spending time but I have absolutely no hint on how to resolve this bug.

### 08/09
**SAR RAM comparison**  
Comparing content from first kernel and second kernel, trying to figure out what registers are to save.

The reason why it hangs might be due to differences in sar ram.

First batch of self comparison (kernel 1 to kernel 1), shows that lines in 6b0X, 6c0X and 6d0X does change, but there is also changes in 6e0X. This might be a factor why things have been different.  
The bank 2 of 7X00 has no changes.

Second batch, kernel 1 and kernel 2, have differences. There are packs where one bit has changed. It might be due to some kind of hardware related, nothing too serious. However, and this was one of my error, the 6a0X has changes I never recorded (like the wakeup address). b0X, c0X and d0X have the usual changes. On batch e0x however, there was no differences in one of the comparison. Therefore it might be hardware related.

Third batch, kernel 1 and kernel 1 bis (recompiled with address offset) at another time, is FULL of differences. It is too hard to make any valuable comparison.

Another concern is the GIC, which is not accessible easily, but surely have differences as well.

Anyway, first, added to backup the value of a04 and a08. It won't change anything, actually, since the value is rewritten.


**Reworking fsw_inst_page**  
Now, I am thinking it does not make sense to save only part of the SAR RAM. I seriously should save the whole thing, instead of some bits. This would require to tweak the whole thing. We can do it this way :
* address, size, blob (up to the next bytem of course)
* address, size, blob ...
This would be the simpliest way.  
In our case, each inst page would require at least 2 pages (which can be easily done of course).

For convenience, inst pages are registered after cmm (0xa0202000).

In new architecture, inst page are at size minimum 1 page. If we go below, there might be  
TODO check fsw_inst_page size

Alllright, it works now. So we moved to a full SAR RAM save restore instead.

Poweroff / reboot still does not work.  

    (SGXCleanupRequest+0x0/0x154)

But Kernel difference seems to work! It was not in vain, few.

No idea whether it is possible to save the GIC. It is very likely that it has things.

There is still conflicts in the wifi side.


**a solution for the wifi?**  
The current problem is if wifi is on in one side, and a switching occurs, the second side will error if wifi is activated. The problem does not occurs when wifi is properly closed then opened again.
```c
<3>[ 134.627075] CFG80211-ERROR) wl_cfg80211_disconnect : Reason 3
<4>[ 134.668548] link down, call cfg80211_disconnected
<6>[ 134.694549] cfg80211: Calling CRDA for country: TW
<6>[ 134.733276] cfg80211: Calling CRDA to update world regulatory domain
<6>[ 134.757415] ADDRCONF(NETDEV_CHANGE): wlan0: link becomes ready
<4>[ 134.799835] wl_android_wifi_off in
<4>[ 134.816345] wifi_set_power = 0
<4>[ 135.342681] =========== WLAN placed in RESET ========
```
Some control is done via wpa_supplicant (most of the beginning lines, actually), this is how android configures wifi user-space side.

woops, wpa_supplicant keeps a file where passwords are in clear : /data/misc/wifi/wpa_supplicant.conf

An easy solution would be to cut it off when switching. I think it is a sensible solution.


**parallel memory unplug**  
This is a complicated scheme. To start the second OS, the SAR RAM must be ready. However, accesses to the SAR RAM are not controlled well. A possibility would be to tamper with the 0x4a32 6c0X range, which is core1 data. Actually, if core1 limits only itself to this range, then we may have something. Currently, memory claiming is the core0 job.

A easy part for this job would be to offline core1 (its state is saved in SAR RAM), to make a backup of its state, replace some data in SAR RAM, make it reach kernel 2 by waking it up (waking up should be a simple command actually). During this time, core0 is forbidden to initiate a suspend (we could use a wakelock for example).

Once in kernel 2, core1 will wake up in a certain path: we need to check whether this path is legal for removing memory.

As a test, we use disable_nonboot_cpus (although we could bring down only one cpu with _cpu_down).

To wake it up to second kernel, arm classic functions are out of question (ie no use of _cpu_up). This is because it will prepare the terrain for bringing up a core, although no core are available. We can, however, use boot_secondary from core0.

At the other side, in core1, it will resume from omap4_cpu_resume, jump to omap4_enter_lowpower() (from arch/arm/mach-omap2/omap4-mpusslowpower.c

A test path would be :
* core0 : to disable_nonboot_cpus, boot secondary, wait for it to sleep again
* core1 : wake up from platform_cpu_die() (called from cpu_die()), and in a normal situation it would jump back to secondary_start_kernel, which we don't want. Let's do some job, then sleep again (??? how).

We can see that a succession of sleep, wakeup, sleep, wakeup can do the job.

Here is the breakdown:
```c
core0
> debugfs test
>> disable_nonboot_cpus()
>> set a flag for memory free
>> boot_secondary() (mach-omap2/omap-smp.c)
>>> the clock of cpu1 will be revived
```

```c
core1
>>>> omap4_cpu_resume()
>>> omap4_enter_lowpower() (mach-omap2/omap4-mpuss-lowpower.c)
>> platform_cpu_die() (mach-omap2/omap-hotplug.c)
> cpu_die: (arch/arm/kernel/smp.c) we do the job and then jump back to...
>> platform_cpu_die()
```

```c
[ 54.010467] Disabling non-boot CPUs ...
[ 54.039978] CORE: cpu_die, die 1
[ 54.043762] CPU1: shutdown
[ 54.056732] CORE1 (test): disabled. Trying to reboot it alone (and I should)
[ 54.082122] CORE: platform_cpu_die, out of low power
[ 54.087127] CORE: cpu_die, reboot 1
[ 54.090637] Let's do some magic things and die again
[ 54.095611] CORE: cpu_die, die 1
[ 55.092651] CORE1 (test): nice, magic things are ongoing
```

It is necessary to do some of the job in cpu_die, otherwise we jump again to secondary_start_kernel, and this is not desirable.
However, we meet the problem of rebooting the kernel again, due to a frozen flag which gets erased.
* when doing disable_non_boot_cpus(), and you MUST not call it again, because it will clear all the frozen_cpus flags, and not set them again (because they set it only when it has successfully offlined it)

So in our case, it goes well except for the fact that it must be done through uart shell, and not adb shell... Wonder why. boot_secondary doesn't like going through adb shell.

Anyway, for the "magic thing", I tried to remove a section via fsw_offline_section. The irq were necessary.
Tried to re-enable irqs, but I think I missed a few things, like masks. secondary_start_kernel does have a few related command.

If not possible, then we would have to go full length (which follow this path)
* cpu_die > secondary_start_kernel > cpu_idle > cpu_die
However, the potential issue is the need for the core0 to acknowledge core1 (we don't have this luxury though).

A curious error: when compiling arch/arm/process.c with my code, the kernel just hangs and does not boot. May it be due to the kernel size?
Well, it has something to do with reading my flag, a system ram address, very often?
I got it, it is something about fsw_base not initialized.
Still, not working as expected.

Current state is:
boot_secondary called from a test debugfs call, which is supposed to call the core1 back to life. core1 is supposed to run normally again via cpu_idle routine, but enter in conflict with core0. I think it is a coherency issue.
Since it is supposed to go via the normal route, we can try and look how enable_nonboot_cpu() does.

Juts reading this, and:
```c
/*
* Wait until the cpu which brought this one up marked it
* online before enabling interrupts. If we don't do that then
* we can end up waking up the softirq thread before this cpu
* reached the active state, which makes the scheduler unhappy
* and schedule the softirq thread on the wrong cpu. This is
* only observable with forced threaded interrupts, but in
* theory it could also happen w/o them. It's just way harder
* to achieve.
*/
```
Looks like it will be a bit complicated to run core1 as a secondary core of the second kernel to make the job of freeing memory. It is a bit hard...

In another hand, it would be worthwhile to find a way to bootup core1 (disable_nonboot_cpu() + boot_secondary()), at least it start. From there, just a flag and CPU1 can jump to the other side, and fake itself as core0, more or less. Or, restore flag0 register SAR RAM content instead of core1 content, looks like it would (almost) work.

Behind, the problem is still there though: in kernel, one cpu is registered as the main core, and



**running anything other than the gnex kernel**  
see http://benbuchacher.com/posts/2012/07/archlinux-on-galaxy-nexus.html


### 09/09
**Jelly bean patching**  
Kernel could accept without problem the patches on jelly bean branch. However, the kernel does not work immediately with ICS (which is not surprising), as the init operation does not seems to do what nexus is expecting.

**Jelly bean file system changes**  
The system is no more the same, therefore it also needs to be mounted beforehand. The order is as such:
```c
userdata
loop system_jb
loop userdata_jb
loop cache_jb
```

In ICS, I had to create some folders between mounting userdata then the loops. This is no more possible here (mount_all will mount all the filesystems in a batch) : About init.rd, there is now a tweak: mount operation is done through the fstab.tuna file, no more in init.tuna.rc (which I would say is an improvement). The command is mount_all /fstab.tuna.

The strategy would require to edit init.tuna.rc to create the /parent/ folder which has access to userdata partition, then to loop mount the rest.

loop@System failed to loop correctly it seems. I think i have bad options. Even when directly put, it has errors.
Oh, now I remember. It is because system.img has a special format which must be converted to a classic one.
* http://forum.xda-developers.com/showthread.php?t=1384600
Reused a previously compiled simg2img, and decompiled a system_jb.img compatible with loop.

So, the issue was not the flag, but more because of system.img not being in a standard format ??

Notice : Surprise though, jb has now a e2fsck! it was time, seriously...


Silly me, so far I was running the old kernel, ha. So anyway, the old kernel won't run JB because of some PVR initialization which has not been done. This is not too surprising, drivers and JB may communicate differently...

So far, this line in fstab.tuna did not work to mount
* /parent/data/media/system_jb.img /system ext4 loop,ro wait
So reverted to the line in init.tuna.rc (like the old way)
* mount ext4 loop@/parent/data/media/system_jb.img /system wait ro

**Jelly bean run**  
Yes, it runs! It is just not possible yet to use mount_all to mount loop devices, same for /cache and /data. Well...

Great achievement. What does it means : we can run (well, at least, if we manage to make switch-suspend work) two separate linux kernels (well, they are also based on 3.0, but they are not strictly the same, so some can embed other features), and it has no effect on the Android version. Looks like it. We however can not (yet) run two different OSes.

Notice: userdata embed a encryption feature which inspect metadata folder.

After some experimentation on JB, it looks more reactive, that is a nice change.


**Jelly bean console and memory**  
UART console not activated 9see debug, but no prompt), usb debugging as well. Brought a few changes in default.prop of the initrd
```c
ro.secure=0
ro.allow.mock.location=0
ro.debuggable=1
```
While the uart console works, now process (acore osmething) don't stop restarting. I suspect tt is because it has not enough memory (it was very limited from the start though... Raised from 20 sections to 22 sections, no problems.


**Attempted jump**  
Failed, badly. it just hangs on the jump. I suspect a bad address being copied.
Something is wrong, I think the fastswitch page is getting rewritten or something, or the instance did not get detected.

Putting memrw, viewmem, busybox and su in jb system.

This would require a little bit investigation on whether we manage to jump or not. It is however quite tiring to jump between two kernels... So, first, make sure we get the proper address back.

Auto suspend works, so I guess it manages to find its way. Then, the problem happens in the master kernel. It can't seem to resume properly.

Trying to jump back and forth between kernel 1 and kernel 2, to determine exactly whether we are jumping back to kernel 1.

One thing to test is whether the classic kernel can restore as well with two less segments : yes, this is not an issue.

So now, kernel 2 when switch suspending will leave a flag, we need to compare it in order to jump back... jumpback address is 0x90073bcc, but it does not jump back. No, it is because the SAR RAM gets restored for ICS, and jumping back to JB it is bound to cause problems, of course. But anyway, it looks like pc reaches 0x800X, so the problem is not here.

Might be because master kernel did not save properly its data on switchboot? There is no reason why the saved data would be incorrect, except for the fact that I did not saved everything.

To check whether the data is correctly loaded in kernel 1 (or at least up to where it works), I need a way to jump back to a safe system so I can properly run. A possible way is for the kernel 1 on resume to set correctly the core flags, the target and the current, so we can resimulate a jump. Let's do this.

Worked (after several trials). Just forgot to set 0xa020001c to 0x12345678.

So, after a few tests, the phone does not hang. It has probably bad values somewhere though. I, however, can not go too far, otherwise MMU is on (occurs just before mmu_on_label), and once it is I can't jump back. I had an idea though, it is to copy the stack pointer (which has in this run the value 0xC7BB9D20 -> 0x87BBX). Later, it is loaded like :

    ldmfd sp!, {r0-r12, pc} @ restore regs and return

Therefore, inspecting the sp gives the return address. And it is indeed correct (return address is C006e74C, in enter_lowpower() where omap4_cpu_suspend is called), nothing wrong on this side. The code (if it gets executed) is not wrong either.

Well, now checking if it gets out of the omap4_cpu_suspend (I doubt it, it did not print what I wanted). Also, it might be due to interrupts not opened, maybe?  
I added some prints, and I don't see them. This might only be due to the fact that uart got powered down or what? It is likely! Damned!  
So there have been a potential conflict when one disabled the UART (kernel 2) and the other did not (kernel 1). When kernel 1 jumps, it does not touch UART state. When kernel 2 jumps, it closes the UART state, and kernel 1... sees nothing.
* !!! *keep KEEP_FIQ_DEBUGGER_ACTUVE or something like that* This is a typical case where the device state is not put in the same state.

And so, that was the missing link. It works, well. Jump between ICS and JB, ha ha. 4 hours on one option...

(in meld, by comparing the memory dump of the sar ram of kernel 1 and 2, there does not seems to be key differences, except for the fact that there is many data I don't understand).


### 14/09
The su binary happened to not work (shell launching a su command, but the shell has no elevated powers). Actually, it was a incorrect chmod, the good value being 06755 9notice the s in the permissions).
* http://androidforums.com/triumph-all-things-root/592189-su-binary-outdated.html

Migration of paper's note to github, using the markdown language.
* https://github.com/krosk/fastswitch-notes


### 17/09
Investigate on which devices are taking time to suspend. Some devices are taking more time than the others.
```c
> omapdss 38.177 msecs
> pvrsrvkm 48.797 msecs
> alarm 14.526 msecs
```
But when reported without prints, the time to suspend is actually quite fast.

    suspend of devices complete after 65.704 msecs 108, ... variable

Those devices are what take the most time. Most of the others are taking a measly 0.030 msecs (the smallest time reported, must be even faster I guess)


Reference time from when suspend start by entering enter_state() (screen might be black from a certain duration already),

    [ 515.804107] PM: Syncing filesystems ... done.

At this point, sync, devices, task are going to be suspended.

And when suspend finish.
Everything is good and going

    [ 516.966888] suspend: exit suspend, ret = 0 (2012-09-17 08:09:34.164855730 UTC)

The process goes like this :
* Switch/suspend request issued B -> early suspend routines (screen becomes black), platform can not be used -(delay)> A suspend operations -> platform suspended -> platform resumes -> resume operations -> platform can be used
* 1.17 1.27 On average, the time is 1.22 second from A to the end. However, the perceived time is slightly longer, because the screen becomes black before the suspend request will put screen black first before doing the proper suspend operations. The time between B and A varies a lot, from 0.2 seconds to a 5-6 of seconds. On the best case, the full time between request and resume is about 1.43 seconds.

Audio is a great problem : if there is sound playing, the platform will not immediately suspend on a kernel issued suspend request, but will suspend if power is pressed (on this case, android will cut the sound, so the kernel can suspend safely). Audio playing will disable the switch

However, a good surprise: as long as both systems have wifi enabled, the switch is valid! or it looks like... the wakelocks are annoying, but it works! As long as the devices have more or less the same state...

The partial suspend is dangerous, in a sense that there is a dependency tree: do not disable devices if its children are still active.


**Debug the power off?**  
When executing power off, the shell does not close completely. We can still use command line, as well as the fiq debugger (and we can reboot from it, that's great!)

Noticed that the /system/bin/surfaceflinger is spinning.

There may be a need to check again how the system is supposed to poweroff.
```c
[ 90.891143] SysRq : Emergency Remount R/O
[ 90.945800] EXT4-fs (mmcblk0p12): re-mounted. Opts: (null)
[ 90.990020] EXT4-fs (mmcblk0p11): re-mounted. Opts: (null)
[ 91.014068] [MODEM_IF] xmm6260_off()
[ 91.027770] Emergency Remount complete
[ 91.255157] PVR: PVRSRVDriverShutdown(pDevice=c78c3400)
[ 91.269775] PVR: SysSystemPrePowerState: Entering state D3
[ 91.293823] PVR: Uninstalling device LISR on IRQ 53 with cookie c7a40800
[ 91.309906] PVR: DisableSystemClocks: Disabling System Clocks
[ 91.374023] Disabling non-boot CPUs ...
[ 91.397827] CPU1: shutdown
```

### 18/09
Lost some notes...

SMC disabled = it works, no problem. Maybe it is a ARM Trustzone problem?

Reversed the emergency remount order, so now a loop device will remount in r/o properly.

**Virtualization and Paravirtualization**  
* http://www.ok-labs.com/solutions/virtualization-and-paravirtualization
Broadly speaking, virtualization technology can support “pure” execution of guest applications, wherein all guest program instructions are handled by the VM platform: privileged instructions are handled by the hypervisor and common ones run natively at full speed.

Alternately, guest program code can be pre-processed (at build-time, load-time, etc.) to remove direct invocation of privileged instructions and insert explicit calls to hypervisor APIs (“hypercalls”). This latter approach streamlines execution by reducing run-time overhead and is termed para-virtualization.
* http://wiki.ok-labs.com/FAQ#IsOKL4analternativetoothercommerciallyprovidedandopensourceoperatingsystemsavailableforuseinembeddedsystemsdevelopment.3F

Said that the OKL4 performance overhead within 4%.
* http://www.slideshare.net/OpenKernelLabs/ok-labs-webinar-android-migration-at-the-speed-of-light
Performance table somewhere, very good results.

So basically, OKL4 is just like Xen, and boxes can be set on top of OKL4. Each box can host a modified OS (via paravirtualization). Linux is the only OS that can be used "natively" (which is actually false: some modifications need to be done).


There might be an interesting feature, it is to catch the events in /dev/input/event2, which capture the button press. With some catch, we could switch instances nicely, he?


### 20/09
**input event handling**
Check both
* http://unix.stackexchange.com/questions/25601/how-do-mouse-events-work-in-linux
* http://linux.die.net/man/3/poll
* http://www.gadgetweb.de/programming/39-how-to-building-your-own-kernel-space-keylogger.html

The current idea would be to push a workqueue or a task with a loop, and poll on the input device?

A user process could do this as well, hey.


The ideal way though would be to have a notifier like operation
* http://lxr.free-electrons.com/source/Documentation/input/notifier.txt?v=3.0

### ->08/10  
Worked on paper

### 09/10
**inventory of potential conferences**
Ref list on draft

**benchmarks**
Made a bunch of benchmark with
* Quadrant: 
* AnTuTu: 
* RandMem:

Irregulars results with Quadrant, as I noticed that the CPU goes down and down... It is due to throttle which will scale down the frequency. Can not disable throttle in linux menu config -> I choose to scale down manually the cpu frequency.

**cpu frequency governor**  
* http://forum.brighthand.com/android-os/286992-cpu-governors-explained-credits-xda-forum.html
Default is interactive, switch between 320Mhz, 700Mhz, 920Mhz, 1200Mhz. Use userspace to limit to 920Mhz (at 1200Mhz it becomes hot).
Use:
```c
    echo userspace > /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
    echo 920000 > /sys/devices/system/cpu/cpu0/cpufreq/scaling_setspeed
```

**restored corrupted filesystem on ICS**  


**Benchmark Results**  
```
Quadrant Pro imm76
CPU  MEM  I/O
4393 1385 975
4375 1439 908
4303 1401 953
4385 1437 900
4374 1319 866
4386 1359 960
4396  860 855
4472 1457 1001
4466 1420 885
4441 1401 946
4352 1355 1040
4474 1444 939


Quadrant Pro omap1 ICS only
4416 1475 893
4476 1470 895
4503 1470 896
4444 1464 964
4497 1395 960
4478 1493 919 163 1441
4490 1397 924 162 1385
4503 1457 1033 162 1378
4501 1417 1032 163 1402
4499 1411 1033 163 1461
4374 1221 1038 162 1389
4480 1385 869 162 1455
4480 1062 968 0 1455
3965 1478 898  0 1428
3961 1362 1053 0 1437
3952 1438 1014 0 1372


omap JB half (fresh system...)
Quadrant Pro 
CPU  MEM  I/O  2D  3D
4478 1451 1307 197 2194
4492 1451 2416 197 2196
4486 1453 2118 196 2204
4497 1441 1982 195 2195
4503 1437 2483 195 2191
4512 1442 2221 195 2228
4506 1150 2507 196 2206
4466 1492 2005 197 2201
4480 1465 2362 197 2193
4476 1444 2359 196 2218

Antutu
RAM  INT   FLO   2D   3D    DAT
823  1387  1043  292  1233  335
771  1383  1044  292  1234  330
825  1394  1045  292  1234  335
923  1251  1004  291  1236  335
818  1342  1018  292  1234  315
785  1325  1031             345
812  1365  1044             330
827  1322  1045             355
823  1336  1046             350
824  1285  971              330


omap ICS half (cleaned but clogged too)
Quadrant Pro
CPU   MEM   I/O   2D   3D
4512  1502  1008  165  1449
4478  1517  1151  165  1347
4471  1513  1145  167  1376
4507  1508  1162  167  1437
4461  1478  1131  165  1453
4468  1509  1138  167  1459
4478  1493  1133  165  1444
4498  1453  1131  167  1403
4496  1543  1179  167  1403
4511  1507  1161  167  1403

Antutu
RAM  INT   FLO   2D   3D    DAT
911  1414  1050  287  1229  355
908  1424  1047  287  1229  365
857  1402  1049  286  1229  360
911  1426  1049  287  1228  375
911  1425  1049  287  1228  385
911  1424  1049  286  1227  390
900  1410  1048  286  1228  365
904  1420  1049  287  1228  380
909  1424  1049  290  1234  380
905  1420  1046  290  1234  365

see mails for randspeed
```


**Quadrant i/o is useless?**  
* http://briefmobile.com/cyanogen-demonstrates-quadrants-flaws
Not really, it makes a repertory somewhere, then make some calculation on it. Curiously, we get curious results by deleting big files on ICS (200+ in score). JB (2gb loop) has stellar scores compared to ICS (16gb native).


**investigate page migration**  
```
offline page breakdown
isolate free pages = make sure they are not allocated 
scan the pfn range, if one is found, do_migrate_range:
  it is removed from the LRU while still in memory. It can be marked as active or unevictable though, depending on its location. Ie, we keep inactive ones.
  if success remove, and the page has no references, added to a page list
    see unmap_and_move() :
 * Obtain the lock on page, remove all ptes and migrate the page
 * to the newly allocated page in newpage.
      try to unmap : remove all ptes
      move to new page : 
        migrate page / migrate page copy
          basically, we replace the old _page cache index_ of the source page by the new _page cache index_ of the destination page. This is how the new page is found??
  page is replaced in the lru at the end
```


### 10/10
**interesting read on hypervisor**  
Comparison between okl4 and vmware mvp.
* http://www.ok-labs.com/blog/entry/vmware-mvp-how-it-works/
Content can be added in related works regarding hypervisor type 1 and type 2.

**related works** 
As well, distinguish ARM v6 and below who have no virtualization support, and ARM v7 and higher which have virt extensions.

**market share**  
good content for the intro of thesis
* http://www.eetimes.com/electronics-news/4229986/ARM-to-continue-tablet-domination-well-into-2013
* http://www.forbes.com/sites/ericsavitz/2012/07/10/tablets-by-the-numbers-its-all-about-apple-and-arm/

**more content in paper**


### 11/10
**Android CDD**  
* http://source.android.com/compatibility/downloads.html

**Sparse memory model perf**  
* http://www.gossamer-threads.com/lists/linux/kernel/842302?do=post_view_threaded#842302

**try RTAS instead of Eurosys**  
* http://www.rtas.org/

**About virtualization on ARM**  
Only the cortex A15 has virtualization extensions. OKL4 has started to play with the simulator provided.
* http://www.ok-labs.com/releases/release/open-kernel-labs-delivers-okl4-mobile-virtualization-for-arm-cortex-a1

**microkernel and hypervisor, differences?**  
* http://conferences.sigcomm.org/sigcomm/2010/papers/apsys/p19.pdf
In two words, hypervisor are a full layer between hardware and guest, and microkernel are a thinner layer.

**codezero**  
Hypervisor for arm devices. Works for galaxy nexus, theyr run concurrently 2 androids.  
Some questions: they say concurrently, one OS on one core. Does that means one OS has only one core available?  
Interesting way of handling storage! Android 0 start an nfs, Android 1 access to it via NFS. This of course requires both instances to run at the same time.  

**memory hotremove**  
One problem is that removed memory is still mapped. It seems, according to the documentation, that memory hotplug never bothers to remove the memmap of removed memory.  
* A new patch seems to be doing it. http://lwn.net/Articles/518689/
* http://thread.gmane.org/gmane.linux.ports.sh.devel/16913

**multi user on android**  
* http://www.zdnet.com/desktop-android-multi-user-android-support-is-on-its-way-7000002143/

**crazy**
* http://onlinelibrary.wiley.com/doi/10.1002/ecjc.20343/pdf
* mint: http://ieeexplore.ieee.org/stamp/stamp.jsp?tp=&arnumber=6103093
* shimos: http://ieeexplore.ieee.org/stamp/stamp.jsp?tp=&arnumber=4591581&tag=1


### 12/10  
**trying the power off trick to unload a system**
in omap/kernel/sys.c, the 
```c
    kernel_power_off()
        kernel_shutdown_prepare()
            blocking_notifier_call_chain()
            usermodehelper_disable()
            device_shutdown()
        disable_non_boot_cpus()
        syscore_shutdown()
        ..
        machine_power_off()
        Question is, does the machine stop here?
    depending on where it is called, it will call
    machine_halt()
        machine_shutdown()
        while(1)
```

In the otehr side, we have on the suspend (kernel/power/suspend.c):
```
    device_suspend()
    suspend_enter()
        disable_nonboot_cpu()
        disable_irqs()
        syscore_suspend()
```

It is important to keep devices in a suspended state, so we really need to do things before kernel_shutdown_prepare().
A tentative would be to use enter_state() (kernel_power_suspend.c).

The strategy would be, if it is not the initial instance, then closing an instance will return the memory to the initial instance (better would be its parent). We need can use the claim framework. 

Woops, failed. That is due to wakelocks. 

Using wakelocks skip. It works. 

Introducing a new mode: TERMINATE.   (and skip wakelocks for this mode). This is an alias of SUSPEND, but introduce one more condition.
Cleaning the instance by erasing the flag on master.
Multiple conditions to claim back memory:
* claim_instance = master_instance (terminal condition to claim memory)
* claim_count = 0 (no need)
* claim_start: use virt_to_phys(PAGE_OFFSET)
* set CMM_CLAIMED flag for each section belonging to instance
* set CMM_AVAILABLE on reclaim

Noticed that we can switch savagely! Corrected this.

Noticed that the memory reclaimed from the kernel space of OS2 is marked as not available. We correct this.

Misc: When we have an error, just skip entirely the switch, how about this?

Noticed a bug, it is when memory offline fails, the suspend will abort but if we retry it will load even if the memory is not free... This is becasue of the script. We need to make a check in the load_file!


**Timing breakdown for instance to instance**
We cheat a little here, we switch from an instance to the same instance. Timing can be done like this
```c
    start_time = jiffies;
    ... activity
    duration = jiffies - start_time;
    msec = jiffies_to_msecs(abs(duration));
    pr_info("%s took %d.%03d seconds\n", label, msec / 1000, msec % 1000);
```
It is handy to track a duration, but there is something odd: the jiffies calculated in the limbo are just... null, which means they are not accounted within the extra low level of the proceeedure. I doubt they are instantaneous...  
There is, however, a mention on Suspended for xxx seconds which seems a reliable source, see kernel/power/suspend_time.c
After reading the source, it is a time registered between the syscore suspend and the ssycore resume. We do not need to track time inside.

What we can do is from the request, to start a jiffie timer, and to report time from the beginning!


**observations**
The next opportunity to suspend. This is the major source of fluctuation. They are all due to wakelocks. 
It can be occaional delays (type 1) or hard delays (type 2). Occasional delays have timeout, and are due to hasard in Android, wifi. Hard delays are due to wakelocks without timeout, ie audio (finishes when it finishes).

Other timings are do not fluctuate so much, the most stable being the limbo (it is a hard sequence, so there is not much to change).
Was afraid that the cpuspeed fluctuation (due to the interactive governor) would change things, but it seems not.

prints reaaaally take some time...

The first time we switch, it is fast, 88 versus 238 ms. There might be some reasons behind... but the code can clearly be optimised, I think?   

Skipping wakelocks will reduce absolutely the delay part. But it will add bug messages on early device (+0.1s), and sometimes lead to a cascade fail of devices suspend. Something i did not notice before, it is the wakelock skip, it is effective for ALL instances, and is disabled each time an OS boots.  
Other bad effect is the audio. It will tear down the wifi as well, if a switch happens when there is a transfer ongoing. The network must reboot. Sometimes, the wifi will crash and reboot itself. It is much more stable if the network is the same between both instances.

Finally, I need to say that skipping wakelocks is altogether a bad idea, notably for the wifi transfers.


**benchmark**
Three cases:
* A 700Mb regular Android
* B 310Mb regular Android (used memblock reserve and remove)
* C 310Mb Fastswitch Android OS1
* D 310Mb Fastswitch Android OS2

Antutu, Quadrant, MFLOPS, and Sunspider

Use:
```c
    echo userspace > /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
    echo 920000 > /sys/devices/system/cpu/cpu0/cpufreq/scaling_setspeed
```

```
Antutu
* A 710Mb regular Android
RAM     CPU1    CPU2    2d      3d      I/O     SDW     SDR
908     1413    1050    288     1228    335     97      186
912     1413    1051    287     1228    365     116     192
907     1412    1050    288     1229    310     129     191
901     1408    1049    286     1229    345     119     190
911     1417    1050    287     1228    370     128     192
* B 310Mb regular Android
RAM     CPU1    CPU2    2d      3d      I/O     SDW     SDR
902     1399    1038    287     1229    365     92      191
902     1404    1040    287     1227    390     97      192
909     1405    1049    287     1229    390     120     191
904     1429    1050    287     1228    385     123     192
903     1400    1050    287     1227    365     122     190
* C 310Mb Fastswitch Android OS1

Quadrant
* A 710Mb regular Android
CPU     MEM     I/O     2D      3D
4485    1400    1114    163     1449
4470    1397    946     162     1420
4512    1410    1135    162     1459
4509    1415    1141    162     1406
4273    1445    1134    162     1449
* B 310Mb regular Android
CPU     MEM     I/O     2D      3D
4500    1491    1002    162     1437
4482    1462    979     160     1458
4337    1410    1128    160     1389
4459    1410    1130    163     1443
4501    1408    1085    163     1461

MFLOPS-v7 
http://www.roylongbottom.org.uk/android%20multithreading%20benchmarks.htm#anchor2
32 Ops/Word for the RAM section
* A 710Mb regular Android
TIME    1T  2T  8T
11.2    426 871 866
11.1    426 874 866
11.2    426 875 869
11.1    425 875 869
11.1    426 868 866
* B 310Mb regular Android
TIME    1T  2T  8T
11.1    425 872 869
11.1    417 865 872
11.1    426 871 868
11.2    425 869 870
11.1    425 874 870

Sunspider
* A 710Mb regular Android 2388.0ms +- 2.1%


```
Just noticed that the PVR bug happened. It may be a bug of the kernel version, and not our problem.