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






Simultaneously try on realview or exynos?