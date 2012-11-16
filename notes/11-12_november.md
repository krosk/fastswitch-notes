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



Simultaneously try on realview or exynos?