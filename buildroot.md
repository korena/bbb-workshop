# Buildroot

Although we're working on the kernel source directly, we're going to use buildroot for:

* building the bootloader (u-boot)
* building a basic filesystem with busybox, along with open source user land packages
* fetching and installing our cross compiler

This approach provides a unified base for all our development needs. We'll have to add the path to the cross compiler to our shell, so we could use it to build the kernel and any other package that we could need later. Eventually, we're going to use buildroot to automate the process of fetching the stable kernel, patching it with all the changes we made, configuring it, building the bootloader and the root filesystem, see how much work we'd have to do if this magnificent tool did not exist??

The advantage of Buildroot over other projects that provide 'recipies' to build stuff is it's extreme simplicity, and the fact that it does not abstract the details of your build components to a level where it becomes very unproductive to debug. Other build systems could have been used here, but in my experience, nothing matches Buildroot in it's reasonably sloped, very short learning curve, and maintainability. When something goes wrong, you'll find that you know enough about the build system to fix almost any issue without significant efforts. 

## Walk through video

### bootloader, filesystem and cross compiler

I explain in the following video, the process of getting buildroot to work for us. This process is set **before** we build the kernel, so this is not our final buildroot configuration for the automated build of our final working system, we're only using it to speed up development and provide a good base to start from.

> Libraries that are needed for actually getting some peripherals to work are not discussed in this 
> video, they're left out so we can discuss them in the right context. This is a short introductory
> video that will show you the very basic setup process.

[![Watch video on Youtube](https://img.youtube.com/vi/8dq8LcYUzcc/1.jpg)](https://youtu.be/8dq8LcYUzcc?t=2090)
