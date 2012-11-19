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