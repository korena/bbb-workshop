# SGX530 driver porting

There's a lot of noisy information out there about the procedure to actually get this thing to work, you need to be very careful about what you follow, else you'd be wasting a lot of time with absolutely no results, believe me, I've been there.

If you're trying to work with kernel 4.x, avoid the following:

* http://processors.wiki.ti.com/index.php/AM35x-OMAP35x_Graphics_SDK_Getting_Started_Guide
* http://software-dl.ti.com/dsps/dsps_public_sw/sdo_sb/targetcontent/gfxsdk/latest/index_FDS.html
* http://processors.wiki.ti.com/index.php/Graphics_SDK_Quick_installation_and_user_guide


Some of the above may be useful for debugging and some general understanding of how the SGX components stack up to provide functionality, but you will have a very hard time trying to forward port the compilable source code for your own kernel, if you value your time, just don't.

## watch the process on youtube:

[![Watch on YouTube](https://img.youtube.com/vi/8nctkhG_afY/1.jpg)](https://youtu.be/8nctkhG_afY)


## getting the SDK

We'll start off by downloading [ti's SDK](http://software-dl.ti.com/processor-sdk-linux/esd/AM335X/latest/index_FDS.html), we're specifically going for AM335x Linux SDK Essentials binary file, this seems to be the only maintained version of the SDK, I may be wrong, do ping me if this is inacurate.

Run the binary, and install it in your project's directory.

```bash
[korena@host:Black]$ chmod +x ti-processor-sdk-linux-am335x-evm-03.00.00.04-Linux-x86-Install.bin
[korena@host:Black]$ ./ti-processor-sdk-linux-am335x-evm-03.00.00.04-Linux-x86-Install.bin
```
when done, you'll find the sdk stuff under whatever directory you specified in the wizard thingy, I set it to ti-sdk.


There are plenty of docs to read within the SDK, a lot of it assumes that you're compiling against the provided ti linux kernel, since we're not going for that, we don't necessarily care about the instructions provided, we'll cd into the directory that contains the Makefile responsible for building the open source shim kernel module for the linux kernel and run our make command there:

```bash
[korena@host:Black]$ cd ti-sdk/board-support/extra-drivers/ti-sgx-ddk-km-1.14.3699939/eurasia_km/eurasiacon/build/linux2/omap_linux/
```


## Setting up

Running make without setting up the environment for cross compiling is of course not going to work, so here's the variables that need to be set before make can work:

* KERNELDIR (Points to the root directory of the kernel you're trying to build the module against)
* CROSS_COMPILE=arm-linux-gnueabihf-
* your cross compiler's bin directory needs to be in the path.

> I found out about KERNELDIR being the pointer to the kernel root directory by reading the INSTALL file in the eurasia_km directory

Since you're building a Module, your kernel (the one you're compiling against) needs to be compiled with Modules support, while you're at it, enable Modules unloading option. I shouldn't have to say this, but you need to actually compile the kernel after configuring it, don't assume configuring the kernel is enough to build a module against it.


## Compiling pvrsrvkm

The process we're going to go through here is likely to change as the kernel stable version evolves, because of the fact that the kenrel's internal APIs change very often, what I will be working with here is Kernel stable version **4.7.4**, which is the stable version at the time of this writing, you can follow the same procedure to compile this module against future kernels, just take the general idea from this post.

### What we know about this Module's source code


We know that this module compiles fine against ti's kernel version 4.4, so it (the module) is familiar with the kernel APIs version that came with that version. We can safely assume that TI's engineers did not make any changes to the internal workings of the kernel, as that is generally not how Corporations go about using the kernel. The divergence from the mainline kernel is (typically) very minor. Most of the changes that we'll see between the mainline kernel version 4.4 and ti's kernel would be in ti's own files within the mainline kernel source, those would be somewhere in arch/arm/mach-omap*/ , and possibly some driver code for their own SoCs' peripherals under /drivers. One place that will almost always have an extra file or two is some directory under the top include directory, since that's primarily where header files are included from when compiling against the kernel.

> If you find yourself adding/changing a header file to the kernel so that some module you're writing/porting would include, there's allways some change in the kernel source code that corresponds to that header file! 

### Resources we're going to use

[lxr.free-electron.com](http://lxr.free-electrons.com/source/) is your best friend, along with git log commands. Of course, we need the source code for both the kernel that the pvrsrvkm module already works with (ti's) and the kernel we're porting the module to (mainline stable 4.7.4).


### General roadmap

We'll start by directly running "make BUILD=release", and wait for compilation to fail, which would give us pointers towards what needs to be done. Simple eh?

> You should always think about changing the Module's source code before touching the kernel source code! the less changes you introduce to the kernel, the easier it is to forward port your module once a new kernel comes out, and the less patching you have to do to future stabe kernels' source code.

### porting

```bash
[korena@host:Black]$ export ARCH=arm
[korena@host:Black]$ export CROSS_COMPILE=arm-linux-gnueabihf-
[korena@host:Black]$ export KERNELDIR=/path/to/project/root/dir/linux
[korena@host:Black]$ export PATH=$PATH:/path/to/cross/compiler/bin
[korena@host:Black]$ export TARGET_PRODUCT=ti335x
[korena@host:Black]$ cd ti-sdk/board-support/extra-drivers/ti-sgx-ddk-km-1.14.3699939/eurasia_km/eurasiacon/build/linux2/omap_linux/
[korena@host:omap_linux$ make BUILD=release
```
The above steps will result in a compilation error, and that would be our first issue to resolve! In my setup, this hits the following error:

```bash
/home/korena/Development/Projects/Beaglebone/Black/ti-sdk/board-support/extra-drivers/ti-sgx-ddk-km-1.14.3699939/eurasia_km/eurasiacon/binary2_omap_linux_release/target/kbuild/services4/srvkm/env/linux/module.c:51:42: fatal error: linux/platform_data/sgx-omap.h: No such file or directory
```

This basically means that the kernel we're compiling against does not have that header file at all. This header file looks like an addition to the kernel by TI, so we're not going to find it in lxr.free-electron.com, this is something we have to get from ti's kernel that is provided with their SDK, so lets copy this file:

```bash
[korena@host:board-support]$ cp ./linux-4.4.12+gitAUTOINC+3639bea54a-g3639bea54a/include/linux/platform_data/sgx-omap.h  ../../linux/include/linux/platform_data/sgx-omap.h
```
> Every time you add a header file that does not define static functions within, there's a corresponding implementation somewhere in your kernel source that makes use of this header file. The reason I'm saying this should-be-obvious fact is that your module would probably compile fine, and you'd think you've won the porting battle, but you'll have inexplicable runtime errors, segmentation falts and possibly kernel panics.

Now that we have the header file, we need to find the source files that are making use of this header file within the ti kernel, we do that by looking through ti's kernel source, using any search capable tool we have:

```bash
[korena@host:linux-4.4.12+gitAUTOINC+3639bea54a-g3639bea54a(processor-sdk-local)]$ git grep 'gfx_sgx_platform_data' -- '*.[ch]'

arch/arm/mach-omap2/pdata-quirks.c:static struct gfx_sgx_platform_data sgx_pdata = {
include/linux/platform_data/sgx-omap.h:struct gfx_sgx_platform_data {
```
There we have it, the file arch/arm/mach-omap2/pdata-quirks.c in ti's kernel makes use of the gfx\_sgx\_platform\_data structure defined in the header file we copied above. The next step is to see if this file actually exists in our own stable kernel, and if it does, what changes does it need to have, this basically means you need to look at the diff between the two files, and figure out what to change.

> I'm using KDE as my desktop environment, and when I need extensive visual asssistance to speed up the process of comparing files, I use Kompare


The following is the diff result for the changes I introduced by comparing ti's kernel and the mainline stable kernel:

```bash
[korena@host:linux(boneblack)]$ git diff HEAD -- arch/arm/mach-omap2/pdata-quirks.c

diff --git a/arch/arm/mach-omap2/pdata-quirks.c b/arch/arm/mach-omap2/pdata-quirks.c
index 6571ad9..5a279ea 100644
--- a/arch/arm/mach-omap2/pdata-quirks.c
+++ b/arch/arm/mach-omap2/pdata-quirks.c
@@ -26,6 +26,7 @@
 #include <linux/platform_data/wkup_m3.h>
 #include <linux/platform_data/pwm_omap_dmtimer.h>
 #include <linux/platform_data/media/ir-rx51.h>
+#include <linux/platform_data/sgx-omap.h>
 #include <plat/dmtimer.h>
 
 #include "common.h"
@@ -48,6 +49,16 @@ struct pdata_init {
 static struct of_dev_auxdata omap_auxdata_lookup[];
 static struct twl4030_gpio_platform_data twl_gpio_auxdata;
 
+
+#if defined(CONFIG_SOC_AM33XX) || defined(CONFIG_SOC_AM43XX)
+static struct gfx_sgx_platform_data sgx_pdata = {
+       .reset_name = "gfx",
+       .assert_reset = omap_device_assert_hardreset,
+       .deassert_reset = omap_device_deassert_hardreset,
+};
+#endif
+
+
 #ifdef CONFIG_MACH_NOKIA_N8X0
 static void __init omap2420_n8x0_legacy_init(void)
 {
@@ -532,6 +543,8 @@ static struct of_dev_auxdata omap_auxdata_lookup[] __initdata = {
                       &omap3_iommu_pdata),
        OF_DEV_AUXDATA("ti,omap3-hsmmc", 0x4809c000, "4809c000.mmc", &mmc_pdata[0]),
        OF_DEV_AUXDATA("ti,omap3-hsmmc", 0x480b4000, "480b4000.mmc", &mmc_pdata[1]),
+       OF_DEV_AUXDATA("ti,am3352-sgx530", 0x56000000, "56000000.sgx",
+                      &sgx_pdata),
        /* Only on am3517 */
        OF_DEV_AUXDATA("ti,davinci_mdio", 0x5c030000, "davinci_mdio.0", NULL),
        OF_DEV_AUXDATA("ti,am3517-emac", 0x5c000000, "davinci_emac.0",

```
I ran make again on the kernel, just so we wouldn't forget.


Now we can go back to our module source and run make BUILD=release again .. which gets us to the error :

```bash
/linux/osfunc.c:3521:17: error: implicit declaration of function 'page_cache_release' [-Werror=implicit-function-declaration] 

page_cache_release(psPage);

```

Lets go ahead and see where this page\_cache\_release(page) is defined, we'll start by looking at kernel 4.4 source code in lxr.free-electron.com, an identifier search for "page\_cache\_release" would tell us that page\_cache\_release(page) is a macro, defined in include/linux/pagemap.h, the macro is defined as put\_page(page). We now have an opportunity to make a very simple change that will not effect the functionality of the module, but will resolve this error and help us move forward. 
If we can find the function put\_page(page) in the source code of the kernel we're trying to port our module to, we can probaly get away with changing every instance in the module's source code from page\_cache\_release(page) to put\_page(page) without influencing the functionality of the module. Looking in lxr again, this time searching (identifier search) in kernel source 4.7, we see that put\_page is defined as a function in include/linux/mm.h, line 751, bingo! We can just call that function directly instead of going through the Macro definition, note that the function resides in include/linux/mm.h header file, which means it's exposed to code that is written against the kernel, this is important, since we can safely include this file (if it isn't already included) without poking too deep at some internal kernel subsystem that is not meant to be exposed as an API.

Do some search and replace in your favourite editor, and replace every instance of page\_cache\_release with put\_page, then rerun make.

the next problem we hit is an issue with get\_user\_pages, used in ti-sdk/board-support/extra-drivers/ti-sgx-ddk-km-1.14.3699939/eurasia_km/eurasiacon/binary2_omap_linux_release/target/kbuild/services4/srvkm/env/linux/osfunc.c the error states that we're making an assignment to pointer without a cast, this is what the compiler thinks, but this generally means that a kernel API changed, keeping the function signature, but changing the argument types, which could indicate a completely different functionality provided by the same function signature, this is slightly tougher than what we faced above, and it could push you down the road of having to actually read the module code and fully understand what the developers are trying to do, then coding your own solution based on the kenrel APIs provided by the new kernel. Because we're lazy, and we'd like to get things done with the least amount of thinking, we're going to first look at the issue at hand, and try to find the easiest way out without breaking stuff. We'll start by looking at the function signature in kenrel 4.4.
lxr tells us that this function is declared in include/linux/mm.h, and that it has the following signature:

```c
long get_user_pages(struct task_struct *tsk, struct mm_struct *mm,
                            unsigned long start, unsigned long nr_pages,
                            int write, int force, struct page **pages,
                            struct vm_area_struct **vmas);
```

So the Module developers are tying to map some physical pages into their virtual address space, or something like that, we won't care unless we have to! So lets look up the same declaration in kernel 4.7.
lxr tells us that this function is still declared in include/linux/mm.h, and that it has the following signature:


```c
long get_user_pages(unsigned long start, unsigned long nr_pages,
                             int write, int force, struct page **pages,
                             struct vm_area_struct **vmas);
```

We can straight away see the problem, this isn't the same API function! And we're getting closer to the edge we'd like to avoid, which is to actually figure out exactly what the module is trying to do, and implementing it ourselves.

> Besides sheer laziness, the reason why you want to avoid modifying the module source code on a functional level is that if you get to the point of re-implementing some part of the module, it most probably will not end there, your changes will introduce more changes, and you'll suffer the dominos effect, potentially for weeks or even months. 

Let's look at two ways we could further investigate this issue:

####  When you're just too lazy to bother(Shame on you)

The first way requires some "intuition", changes are rarely so radical that they would break such an integral part of the kernel as the memory management subsystem, we can assume that this change of functionality is not a real game changer, and look for an API that matches, or almost matches the API we're looking for, in our case, we find the following declaration in kernel 4.7:

```c
long __get_user_pages_unlocked(struct task_struct *tsk, struct mm_struct *mm,
                            unsigned long start, unsigned long nr_pages,
                            int write, int force, struct page **pages,
                            unsigned int gup_flags);
```
The first thing you'll notice is the double underscores at the beginning of the function, this is not good, as it indicates that its for internal use, but gives us 'hope', because it is so close to the function we're looking for, with the exception of the last argument, you may be tempted to just use this funciton and move on, especially because the module w're working on is actually passing "NULL" to the last argument, this is a bad idea, and you should know better. Another function that is very promissing in kernel 4.7 is:

```c
long get_user_pages_remote(struct task_struct *tsk, struct mm_struct *mm,
                             unsigned long start, unsigned long nr_pages,
                             int write, int force, struct page **pages,
                             struct vm_area_struct **vmas);
```

This, is an exact match! At this point, you could just jump in and use this function (Don't !), as it provides exactly the same interface to our module, this could still have undesirable side effects, but the odds could "hopefully" be in your favor.

> I've actually went ahead and tested this lazy way, and things worked ok, but it really isn't a good thing to do, it may have worked out OK in his instance, but it could have failed, and introduced some unpredictable issues down the line.


#### A closer, more careful look

Use git log and read the commit messages that actually introduced this change.
Let's go ahead and do an identifier search on lxr for the funciton get\_user\_pages in kernel source 4.4.
Instead of looking for definitions, we look at references, preferably by drivers that are closer to our own module, namely, gpu drivers. We see that drivers/gpu/drm/via/via_dmablit.c, line 242 references this function in kernel source 4.4:

```c
ret = get_user_pages(current, current->mm,
                              (unsigned long)xfer->mem_addr,
                              vsg->num_pages,
                              (vsg->direction == DMA_FROM_DEVICE),
                              0, vsg->pages, NULL);
```

Let's see what happened to this reference in kernel source 4.7 (lxr.free-electron.com):


```c
 ret = get_user_pages((unsigned long)xfer->mem_addr,
                              vsg->num_pages,
                              (vsg->direction == DMA_FROM_DEVICE),
                              0, vsg->pages, NULL);
```

The new implementation simply drops the stuff in task_struct (first two arguments), so they're no longer needed! Let's actually investigate the commit messages that introduced this change, cd into the linux kernel source (the stable 4.7.4 kernel we're building against), and:

```bash
[korena@host:linux(boneblack)]$ git log -- drivers/gpu/drm/via/via_dmablit.c
```

scrolling down, we can see that the commit message says the following:

```bash
commit d4edcf0d56958db0aca0196314ca38a5e730ea92
Author: Dave Hansen <dave.hansen@linux.intel.com>
Date:   Fri Feb 12 13:01:56 2016 -0800

    mm/gup: Switch all callers of get_user_pages() to not pass tsk/mm
    
    We will soon modify the vanilla get_user_pages() so it can no
    longer be used on mm/tasks other than 'current/current->mm',
    which is by far the most common way it is called.  For now,
    we allow the old-style calls, but warn when they are used.
    (implemented in previous patch)
    
    This patch switches all callers of:
    
            get_user_pages()
            get_user_pages_unlocked()
            get_user_pages_locked()
    
    to stop passing tsk/mm so they will no longer see the warnings.
    
    Signed-off-by: Dave Hansen <dave.hansen@linux.intel.com>
    Reviewed-by: Thomas Gleixner <tglx@linutronix.de>
    Cc: Andrea Arcangeli <aarcange@redhat.com>
    Cc: Andrew Morton <akpm@linux-foundation.org>
    Cc: Andy Lutomirski <luto@amacapital.net>
    Cc: Borislav Petkov <bp@alien8.de>
    Cc: Brian Gerst <brgerst@gmail.com>
    Cc: Dave Hansen <dave@sr71.net>
    Cc: Denys Vlasenko <dvlasenk@redhat.com>
    Cc: H. Peter Anvin <hpa@zytor.com>
    Cc: Kirill A. Shutemov <kirill.shutemov@linux.intel.com>
    Cc: Linus Torvalds <torvalds@linux-foundation.org>
    Cc: Naoya Horiguchi <n-horiguchi@ah.jp.nec.com>
    Cc: Peter Zijlstra <peterz@infradead.org>
    Cc: Rik van Riel <riel@redhat.com>
    Cc: Srikar Dronamraju <srikar@linux.vnet.ibm.com>
    Cc: Vlastimil Babka <vbabka@suse.cz>
    Cc: jack@suse.cz
    Cc: linux-mm@kvack.org
    Link: http://lkml.kernel.org/r/20160212210156.113E9407@viggo.jf.intel.com
    Signed-off-by: Ingo Molnar <mingo@kernel.org>

```
Bingo!, let's just drop the first two arguments from our call to get\_user\_pages,and move on.

> We're still talking about file: ti-sdk/board-support/extra-drivers/ti-sgx-ddk-km-1.14.3699939/eurasia_km/eurasiacon/binary2_omap_linux_release/target/kbuild/services4/srvkm/env/linux/osfunc.c




Running make BUILD=release again takes us to the next issue:

```bash
/home/korena/Development/Projects/Beaglebone/Black/ti-sdk/board-support/extra-drivers/ti-sgx-ddk-km-1.14.3699939/eurasia_km/eurasiacon/binary2_omap_linux_release/target/kbuild/services4/srvkm/env/linux/mmap.c: In function 'DoMapToUser':
/home/korena/Development/Projects/Beaglebone/Black/ti-sdk/board-support/extra-drivers/ti-sgx-ddk-km-1.14.3699939/eurasia_km/eurasiacon/binary2_omap_linux_release/target/kbuild/services4/srvkm/env/linux/mmap.c:790:47: error: incompatible type for argument 3 of 'vm_insert_mixed'
    result = vm_insert_mixed(ps_vma, ulVMAPos, pfn);
                                               ^
In file included from /home/korena/Development/Projects/Beaglebone/Black/ti-sdk/board-support/extra-drivers/ti-sgx-ddk-km-1.14.3699939/eurasia_km/eurasiacon/binary2_omap_linux_release/target/kbuild/services4/srvkm/env/linux/mmap.c:50:0:
include/linux/mm.h:2165:5: note: expected 'pfn_t {aka struct <anonymous>}' but argument is of type 'IMG_UINTPTR_T {aka long unsigned int}'
 int vm_insert_mixed(struct vm_area_struct *vma, unsigned long addr,
```

This is another instance of an API change, following the same investigative procedure above, we come to know that in kernel 4.4, vm\_insert\_mixed's last argument was of type unsigned long (in include/linux/mm.h):

```c
int vm_insert_mixed(struct vm_area_struct *vma, unsigned long addr,
                         unsigned long pfn);
```

But Kernel 4.7 changed it to :


```c
int vm_insert_mixed(struct vm_area_struct *vma, unsigned long addr,
                         pfn_t pfn);
```

Not a big deal, let's look for pfn_t definition in kernel 4.7:

```c
/*
 * pfn_t: encapsulates a page-frame number that is optionally backed
 * by memmap (struct page).  Whether a pfn_t has a 'struct page'
 * backing is indicated by flags in the high bits of the value.
*/
typedef struct {
        u64 val;
} pfn_t;
```

Ok, let's just introduce this change to file /home/korena/Development/Projects/Beaglebone/Black/ti-sdk/board-support/extra-drivers/ti-sgx-ddk-km-1.14.3699939/eurasia_km/eurasiacon/binary2_omap_linux_release/target/kbuild/services4/srvkm/env/linux/mmap.c , by adding our own definition of pfn_t and matching the new API.

```c
IMG_UINTPTR_T pfn; // existing pfn declaration
pfn_t kpfn; // our added bit

[...]

// at the point where pfn is assigned :
pfn =  LinuxMemAreaToCpuPFN(psLinuxMemArea, uiAdjustedPA);
kpfn.val = (unsigned long) pfn;  // our code

// at the point where vm_insert_mixed is called:
result = vm_insert_mixed(ps_vma, ulVMAPos, kpfn); // our kpfn instead of pfn

```

Go ahead and make BUILD=release again, this resulted in full compilation without any issues, if you face more trouble, just go ahead and hack at it, you know what to do!


> using BUILD=debug would compile fine, maybe you'd have to fix some more stuff, but it works exactly the same, the issue with that is the fact that later, when you're trying to load the user land binaries, the script that is provided with the SDK, which is responsible for loading the kernel module pvrsrvkm, and 'gluing' it to user space binaries would complain that the compilation options between the user land modules and the kernel land modules don't match, and it'll just fail. Maybe there's a way to get debug-enabled user land blobs somewhere, but I don't know where that is.

### user land blobs

Now that we have our pvrsrvkm.ko successfully compiled against the kernel, let's move on to fetch the binary blobs that are needed to actually operate the GPU chip. I've looked around the interwebs for a single place where I can pull all the required binaries and just dropping all of them into my root filesystem's /usr/lib and /usr/bin directories, but that proved very difficult as there's just too much noise out there, so what we're going to do here is copying the necessary libraries from within our ti SDK. The basic libraries and binaries that are needed to operate this thing can be found in the provided sysroot under ti-sdk/linux-devkit/sysroots/cortexa8hf-neon-linux-gnueabi, but the problem with this sysroot is the obvious bloat of libraries that we do not really need to get our GPU to work, so instead of working against this sysroot , we'll be adding bits and pieces, responding to every failure we face at runtime, and adding only the necessary libraries to get the driver to bootstrap and do our bidding. The final, minimal filesystem that We require to have the driver successfully register into the kernel and not complain about missing stuff can be found [Here](TODO: link to the basic filesystem that only gets the driver to bootstrap) 

This is a pretty involved process, and it would take ages to document in writing, so Here's a video that walks you through the process:


### testing functionality

The way we'll go about testing functionality is by downloading Imagination technology's SDK, compiling some of their examples, running them in our BBB board, and monitoring the usage of CPU and GPU. So let's get to it!


#### getting PowerVR SDK

Head over to [PowerVR SDK Downloads](https://community.imgtec.com/developers/powervr/installers/) and download the binary for your system (host machine, of course).
run the downloaded installer, The only components we need for our purpose are :

* PVRTune under PowerVR Tools
* Native SDK under PowerVR SDK

> This thing is going to ask for root privelages, just fakeroot it, some things will fail, but they're not really important.


Once we have Imagination technology's SDK extracted, we'll navigate to examples directory, and start with the beginner level, we're going to compile the helloAPI example, then we'll move to advanced examples and compile ExampleUI example. Read the getting started documentation under SDK/Documentation/SDKBrowser and figure out how that's done, or just watch the video:







