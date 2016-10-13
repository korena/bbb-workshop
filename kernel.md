# Linux Kernel 

We're going to be working with the mainline linux kernel for this project, this file gives you pointers in a relatively easy to follow steps to get from cloning the kernel, to patching it for the purpose of working with the beaglebone black without the need for any distro or specific build system.

---
 
## Linux Kernel cloning

---

The Latest *stable* kernel today (Wednesday,September 21, 2016) is **version 4.7.4**. You may be tempted to go for a shallow clone because of the not\-so\-bandwidth\-and\-storage friendly size of cloning everything, but at least have a proper depth that will include a reasonable number of commits back until the Kernel you know works for you. Of course, a better approach is to have a locally available clone of the kernel (working copy), pointing your origin at a proper source, and use it for all your linux projects with clever branching and [Out of tree builds](http://www.crashcourse.ca/wiki/index.php/Building_kernel_out_of_tree). 

*cloning*:

```bash
[korena@host:Project]$ git clone git://git.kernel.org/pub/scm/linux/kernel/git/stable/linux-stable.git

```
> If we intended to work with some fast paced, bleeding edge stuff,
> we could have cloned from https://github.com/torvalds/linux.git,
> but in our context, both would work the same, since we're targeting 
> the *already* stable kernel, we may have had some differences in terms
> of tags and stuff, but whatever we're doing here, could be done against
> a clone from torvald's tree.
 

Once you have your kernel, cd into the top directory, and create a branch out of the stable tag:

```bash
[korena@host:Project]$ cd linux 
[korena@host:linux (master)]$ git checkout -b boneblack v4.7.4
Switched to a new branch 'boneblack'
[korena@host:linux (boneblack)]$
```

You are now ready to configure your kernel for the beaglebone black.

> having your command prompt show the branch in which you're operating,
> along with coloring,user ,host and directory, is a matter of adding
> the following to your ~/.bashrc:
> ```bash
>function parse_git_branch () {                                                     
>   git branch 2> /dev/null | sed -e '/^[^*]/d' -e 's/* \(.*\)/(\1)/'               
>}  
>BLUE="\[\033[1;34m\]"                                                              
>GREEN="\[\033[1;32m\]"                                                             
>RED="\[\033[0;31m\]"                                                               
>YELLOW="\[\033[1;33m\]"                                                            
>NO_COLOR="\[\033[0m\]"                                                             
>PS1="[$GREEN\u@$BLUE\h$NO_COLOR:\W$YELLOW\$(parse_git_branch)$NO_COLOR]\$ "
> ```

## Linux Kernel configuration

Now that we have a fresh working copy of the stable linux kernel, we'll go ahead and configure it to work with the BBB, before we get to configuration, we need to set the ARCH environment variable, I'll try to stay away from environment variables when working with the kernel, as I believe explicit make command line options are clearer, you may of course 
write a bash script and source it, but we're not going to do that for simple options.

The command we're going to use to configure the kernel is:

```bash
[korena@host:linux (boneblack)]$ make ARCH=arm menuconfig
```

> If you miss the above ARCH=arm option, menuconfig will come up,
> but it will default to a different ARCH (your machine's), and 
> you won't see the options specific to ARM processors.

Configuring the kernel is a matter of being familiar with the hardware at hand. We know that the BBB board has a Cortex-A8 AM335x SoC, so we probably want all AM335x configs in the kernel to be enabled, we also want all the drivers corresponding to the other chips available on the board to be set. The main challenge with this process is the fact that, some SoC vendors reuse internal peripherals between different chips, and drivers that are specific to a certain peripheral might depend on the user setting a configuration that is not specifically under the SoC in use. There's also the problem of not being entirely familiar with every IP within the SoC you're using, or having subsystems within an SoC that do not carry the exact name of the corresponding kernel config entry. To work around those issues, the best course of action, if available, is to use a defconfig, which is generally a sane configuration starting point often provided by the vendor of the SoC or the Board you're using. This is not strictly necessary to get things working, but you will spend a treamendous amount of time getting things to work if you have no access to a defconfig, the way I configured this kernel is by carefully going through the configuration of the ti provided linux kernel with version 4.4, slowly checking the differences and updating accordingly. I ended up with a somewhat bloated kernel, as many features are not needed but compiled in anyways, for fear of breaking things, when I'm done with getting all the subsystems to work the way I expect them, I could go back and carefully trim my configuration down to a smaller size, which would result in a smaller kernel. Knowing myself, that will probably never happen. I've exported my configurations for this project into the file [kblack_defconfig(TODO: link)](https://github.com/korena/bbb_dashboard/blob/master/linux/arch/arm/configs/kblack_defconfig), you can use it as your starting point. to load this config file, use:

```bash
[korena@host:linux (boneblack)]$ make ARCH=arm kblack_defconfig
```

Then work your way through menuconfig and edit as you see fit.

> You can search for a specific config entry in menuconfig by hitting / key on your keyboard,
> which will bring up a prompt where you can enter your search query, when you press return,
> the results will be listed if found, every listed result will have a number next to it
> (in brackets), hitting this number on your keyboard will take you to that result.


## Device tree source

The device tree that corresponds to the BBB board is am335x-boneblack.dts, we're going to edit this file directly to arrive at the setup we'd like to achieve, then create a patch against the unmodified stable kernel so we could reproduce our setup with a clean mainline kernel instead of using a modified kernel tree, this is generally the way you should use the kernel.

```bash
[korena@host:linux (boneblack)]$ vim arch/arm/boot/dts/am335x-boneblack.dts
```
You can follow the commit history of this file to figure out how it changed throughout the development process to arrive at whatever state it's in right now:

```bash
[korena@host:linux (boneblack)]$ git log -- arch/arm/boot/dts/am335x-boneblack.dts
```
The verbose commit messages should be enough to understand the changes that came with each commit.


## Patching

When you're working on the boneblack branch we created above, directly on the kernel source, and so you don't need to do any patching, but when you're changes are final, and you'd like to have a distributable source, you should create a patch between your latest commit in this branch (boneblack), and the base commit off of which you started development, in our case, the commit pointed at by the tag *v4.7.4* . It's generally recommended to split your patches into distinct units, for example, a patch that only introduces changes to the dts file, another patch that adds files that are entirely new, a third that introduces changes to existing kernel source files. This will allow a level of granuality that would help you if you're moving forward in a rolling distro-like setup, where you're keeping your kernel up to date, and following stable kernel releases closely. This is not always useful, but separating the device tree changes is almost always necessary, if you change something in your device tree, and would like to use the same patches that modified the kernel source, but change only the enabled devices/ configurations of your pinctrl or clocking signals, then you want to change only the device tree, and apply patches that change only the dts file. 


> An external patch was needed before I could get the PWM subsystem to function, which was necessary to control the DRM enabled driver (backlight is PWM driven), namely : https://patchwork.kernel.org/patch/9005841/, this patch is not yet in the kenrel, so the state of the dts files related to the am335x SoCs does not match the backing drivers withing kernel 4.7.4, the next code listing demonstrates patching using this patch.

```bash
[korena@korena-solid:linux(boneblack)]$ patch -p1 < ../external\ patches/pwm-clock-fix.patch
patching file drivers/clk/ti/clk-33xx.c
patching file drivers/clk/ti/clk-43xx.c
Hunk #1 succeeded at 115 with fuzz 1.

```


## Cross Compiing

Once you're satisfied with you configuration, build the kernel and the dtb (device tree blob) file using:


```bash
[korena@host:linux (boneblack)]$ make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf-
```
this will produce the following files:

+ arch/arm/boot/zImage
+ arch/arm/boot/dts/am335x-boneblack.dtb
