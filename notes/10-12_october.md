# Daily notes on fastswitch
Roughly edited document research notes
Author: Alexis Hé

***

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
907     1424    1050    287     1229    345     96      190
901     1425    1051    287     1229    350     119     192
904     1422    1050    274     1228    365     125     191
908     1422    1049    286     1228    355     122     189
898     1418    1039    286     1228    345     123     190
* D 310Mb Fastswitch Android OS2
891     1404    1046    287     1228    215
904     1421    1047    286     1229    295
903     1420    1045    286     1227    305
903     1416    1046    285     1227    305
901     1419    1045                    280


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
* C 310Mb Fastswitch Android OS1
CPU     MEM     I/O     2D      3D
4514    1432    1108    163     1408
4448    1413    1113    159     1438
4529    1453    1129    162     1455
4266    1355    1133    163     1435
4530    1420    1137    163     1453
* D 310Mb Fastswitch Android OS2
4416    1347    2418    162     1454
4479    1517    2233    160     1459
4499    1465    2334    160     1453
4498    1478    2298    160     1452
4499    1470    2070    160     1444


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
* C 310Mb Fastswitch Android OS1
11.1    422 871 871
11.2    426 872 826
11.1    426 874 871
11.1    424 859 867
11.1    426 869 869
* D 310Mb Fastswitch Android OS2
11.7    416 810 812
11.4    420 835 832
11.4    421 820 858
11.5    416 840 840
11.4    422 801 845

Sunspider
* A 710Mb regular Android           2388.0ms +- 2.1%
* C 310Mb Fastswitch Android OS1    2288.0ms +- 1.2%
* D 310Mb Fastswitch Android OS2    2364.0ms +- 2.7%

```
Just noticed that the PVR bug happened. It may be a bug of the kernel version, and not our problem.


### 13/10
That was hard, but finally found the correct Latex template to use for IEEE Computer Society
* http://www.computer.org/portal/web/cscps/submission#pages

* http://www.ifi.uzh.ch/icse2012/how-to-submit/  
I think i need usletter format, and the compsocconf attribute.

### 14-16/10
Got latex working.

Finished the paper.

Nothing to say, really. Word was good for making figures and exporting them to pdf.

### 17/10
Update Android SDK

Nexus 7 lost its root. Packing a unix environment on windows with:
* https://github.com/bmatzelle/gow/wiki

Went back to Ubuntu.  

Process to root back the Nexus 7:
* adb reboot-bootloader
* fastboot flash system system.img (wasn't sure of what there was left on device)
* got scripts from http://android-dls.com/wiki/index.php?title=HOWTO:_Unpack,_Edit,_and_Re-Pack_Boot_Images
* git mkbootimg from http://code.google.com/p/zen-droid/downloads/detail?name=mkbootimg
* unpack.pl boot.img
* in the ramdisk image, edit default.prop and make the ramdisk non secure
* repack.pl boot_alt.img
* fastboot boot boot_alt.img
* check root access with adb root
* adb remount
* get a su binary somewhere http://forum.xda-developers.com/showthread.php?t=1538053 (download the CWM super su package, there is a binary inside
* adb push su /system/xbin/
* adb shell
* chmod 4755 /system/xbin/su
* If running root checker, the device should be rooted from this point
* Install supersu from google play
* reboot normally  
The result should be a secure device (can't root with adb root). boot.img is the stock one.

**Refactor code**
Noticed that in the omap kernel, the branch for ICS (android-omap-tuna-3.0-ics-mr1) has frozen. Our implementation is currently based on this branch, so there is no need to tamper with it anymore.  
I consider moving to the JB branch altogether, so it becomes the initial system.  

The branch android-omap-tuna-3.0-mr0 is not compatible with ICS, I suspect it is too new, or not adapted to the requirements of ICS (wrong binaries or whatever).

**show history of a file**
```
git log <file>
```

**show commit content**
```
git show <hash>
```

**compiling jelly bean**
It may have not happened before, but using ics defconfig makes the wifi act up (up, reset, up, reset). I copied a new def config from the previous jelly bean port, and it works, but I don't know what I changed.

The working branches with a clean dejllybean system, and a proper config, are:
* android-omap-tuna-3.0-jb-mr0, most recent version september 2012
* android-omap-tuna-3.0-jb-pre1, version june 2012 (pre-release I guess, it stopped more or less)
* android-omap-tuna-3.0-mr0(.1) deprecated (Nov 2011)
* android-omap-tuna-3.0 deprecated 2011
* android-omap-3.0 updated september 2012
From this distribution, what I see is that the tuna board has no more need of any suport, the lastest with support being the jellybean one. ICS stoppsed on April.

**Working on which next platform?**
x86 was not a bad idea: we have qemu, linaro kernels, uboot, and mostly anything possible. Wakelocks have disappeared, but still...  
I am so stupid, linaro is doing ARM only work... For x86, just get the regular kernel, ha!
Aww, forget it, I will stay on ARM, I have no time to do other things. However, I will migrate to versatile express.


**Refactor 1: initrd relocation**
Separated initrd relocation on two patchs, one handles the kernel code, the other handles the ARM specifc part.

**Refactor 2: memory hotplug**
It is a bit complicated here, sparsemem and memory hotplug are separate piece of code.  
First, we do sparsemem. 
Each sub-architecture has a ARCH_SPARSEMEM_ENABLE to indicate it supports. for OMAP4, we can do the same. Sadly, currently only tuna supports it, so it is a bit complicated to make a generality? Maybe put this on MACH_TUNA instead.

So anyway, first is to set ARCH_FLATMEM_ENABLE by default. Then add sparsemem as an option. Then add memory hotplug. support. Then ARCH_POPULATES_NODE_MAP.

By doing so, we have a proper sparsemem functional.

**refactor fastswitch**
Got a lot of work here...  
First, make a proper directory in the linux tree, called muxos.  Found that the Kconfigs work in a tree of inclusions. Only one Kconfig is launched at the beginning (arch/arm/Kconfig) then every thing is added via source, tu build the Kconfig menu. So for an architecture to support muxos, there is only the need to add 'source "muxos/Kconfig"'.

Steps:
* import the muxos header
* import the debug fs; it is something relatively stable and can be in a separate file.
* import the interface to set sections offline/online
* import pretty much anything regarding memory initialization
* import the functions to load a file
* (boot step) make the change in the decompressor head + the suspend body
* (switch step) make the relative changes
* (transfer step)
* (closing step)

And don't rewrite things! they are already done, I should just refactor, not recode the damn whole architecture...

### 18/10
**refactor**  
On architecture dependant functions, I count those:
* image_head_check: check the existence of the fsw_page, the current action, and the addresses of tags pointer and image location. Belongs to BOOT_STEP  
... to finish later, there are more urgent things.

**graduation procedure**  
Deadline is 16th november.  

From Vincent thesis I see:
* Abstract
* Summary
* Introduction
* Related works (can write a lot here, one subsection for any category) _ 8 pages
* Description of the system _ 30 pages
* Experiment _ about how they tested the system. I have no data on this though. _ 8 pages
* Results and analysis _ about how we can make things better _ 5 pages
* Conclusion

From Michel thesis I see:
* Abstract
* Introduction
* Review of all kinds of soft doing the same thing
... there is someting wrong

From MetaOS I see:
* Abstract
* Introduction _ 5 pages
* Related works _ 7 pages
* Design _ 17 pages
* Implementation _ 30 pages
* Experiment _ 5 pages
* Conclusion + future works _ 1-2 pages

MetaOS is very similar to what we done, let's start from here.

**Plan**  
Abstract, in chinese, we will look at it later.

* Introduction: our introduction of the paper is a good base. But we need to indicate more data on:
```
- current landscape of mobile device with the ARM domination, and the reasons the usage model is changing
- how supporting multiple OSes is a solution, and how it solved it on other platforms
- how those solutions are going to come to mobile devices
```

* Related works: just introducing the major works, one page for each? Don't need to talk about their high and their low, we do this in Design.
```
- OS switching
- Multi-boot on Android
- Virtualization / parvirtualization
- Exemple of Cells
```

* Design: 
```
- The problem: running multiple OSes blabla
    Presenting the general challenges in running multiple OSes: two paradigms with the parallel execution / the sequential execution. Their high and their low, each regarding OS state, device state, memory state, isolation, 
- The solution adopted: MuxOS
    Design choice, why this design, how it solve them
    - Sequential execution
    - Integration in kernel
```

* Implementation
```
- Execution state:
    Suspend-and-resume
    OS
    CPU
- Devices
    Suspend-and-resume
    Each device case
    ...
- Memory
    Memory hotplug
    Memory division
```


* Experiment:
```
- The platform
- The methodology
    Performance    
```

* Results and analysis:
```
- Results
```


**THUTHESIS**  
is a real pain in the ass. Not compatible at all with windows, and Miktex, so... 
* https://code.google.com/p/thuthesis/

**removed packages**
No idea if they are necessary or not, but according to this link
* 
it is legal to remove ccmap, and according to this one
* 
it is okay to remove hypernat because it is already included in hyperref.

**zhmetrics and xetex and xeCJK**
This package enables the use of fonts, although it does not embed them with it. Must find a way to support chinese fonts within miktyex.
* http://miktex.org/packages/zhmetrics

Got some font troubles, wanted to use xetex which is able to use the fonts installed in the system, but that's a bit complicated again... 
```
l.52 \char_make_active:n
```

* http://docs.miktex.org/2.7/relnotes/#id678352
According to this doc, xetex is already included in miktex. The tool to use is therefore xelatex or luatex. pdflatex does not work anymore. 

Tried to update miktex. It is starting to look good. Now they report I lack the Adobe Song Std  font. Got them here:
* http://ishare.iask.sina.com.cn/f/15105086.html 

Got also a problem of 
```
xeCJK error: "key-unknown"
```
And no solutions yet.
* http://www3.newsmth.net/nForum/#!article/TeX/313232
I ended up editing thuthesis.cls, looking for ItalicFont and changing to SlantFont.

Trying to enable xeCJK separately.
* http://ftp.ctex.org/mirrors/CTAN/macros/xetex/latex/xecjk/xeCJK.pdf
So curiously, the Adobe fonts did not work. I replaced them by SimSun, SimHei, FangSong in thuthesis.cls and finally i've got some words on pdf! 
Damn. Anyway, in page 9 of the previous pdf, here is the equivalent between the chinese name and the name in English. 

### 19/10
**Working with THUthesis**
Started an initial draft, putting the plan inside. Planning to start a git repo... why not?

### 22/10
**Separated the notes**
They were getting very slow to edit...

**Revised plan**
Our project has the difficulty of distinguishing design and implementation. Both are tied together, because I rely on Linux centric features to design how things are going to be. 

The current paper has this structure:
```
Architecture
    Principles
        Execution priority
        Device state conservation
        Usage flexibility
    CPU and Execution state
    Physical Memory
    Devices
    Storage
    Isolation
Implementation of key features
    Implementation overview
    Booting OS
    Switching OS
    Transferring memory
    Exiting OS
Evaluation
    Usability
    Performance
    Timing
    Power Consumption
Conclusion
```

A rough silu of how MuxOS has been designed, with the incremental changes:
* All OS are on RAM. Because it is good, and quick to go from one to another, and avoid transfer time if we do OS migration.
* a) One OS has full control of the hardware, the other OSes sleep. Because one user can use everything on the platform without limitations. 
* b) Of course, must be able to go from one to another OS, and not lose the OS state + device state.
* How to do a) and b)? (implementation?) Use of suspend-to-ram to suspend the others OS. We are able to switch from one to another like this, but we can't test it yet.
* Many operations to do, many values to keep: we decided to reserve a common memory accesible to all instances, called FSW_PAGE, and its structure changed during the work.

Booting up a new kernel (static memory):  
* Moving a kernel to another start address requires to provide mem arguments and add PHYS_TO_VIRT patch.
* We started by setting up a static physical memory layout (dividing the RAM by two) and compiling two distinct kernels.
* Booting up a new kernel requires to load the initrd, the zImage, and prepare some ATAGS. Used the /dev/mem interface, and copied the data to arbitrary addresses (initrd must conform to the kernel boot args, for example). NOTE: we can do something dynamic here, actually, by altering the ATAGS.
* Then we made a trampoline code at the end of suspend by making a warm-reset. This is because the second kernel requires an initialized environment.
* It is also necessary to add a trampoline code in the decompressor head, to jump to the second kernel. The target address has been retrieved from FSW.
* There are some impact on initial kernel:
* a) Since the bootloader will reload, the first kernel must reserve the modified zones (including zImage, initrd, bootloader, atags, ...). Those zones were detected beforehand by making a copy of the system and comparing the copy with the original (overwritten by bootloader).
* b) This problem forces us to relocate initrd, since it gets extracted in place and gets rewritten in the process. zImage does not get extracted in place, so it is ok.
* c) The state of the first OS is saved on SAR, but we did not backup it: solved at the next step.

Switching instances:
* The suspend path has been altered, in order to check whether a switch order has been issued or not. Some code make checks before and after the suspend and resume.
* The code to switch has been placed at the very beginning of the resume path (placed here because the MMU is disabled on this location). This includes code to backup the source context (SAR RAM content) into FSW, replace it by the destination context, and jump to the first instance resume address. The backup and retrieve are done from FSW.
* This of course requires to know what is the jump address. It is omap4_cpu_resume on our case, and we need to register it in FSW as well.
* 

* However, that does not answer how we plan to share RAM.
