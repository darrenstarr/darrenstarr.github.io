---
layout: post
title: "Writing a Linux Kernel Module on Windows (Part 1)"
author: "Darren R. Starr"
categories: article
tags: [article,linux,kernel,module]
image: cliffs-mountains-nature-42154.jpg
---

It's be a really long time since I've broached the subject of kernel development on Linux. In fact, I hardly recognized the kernel code when I started this project.

## The goal of the project.

Let's start with what I was trying to accomplish. I wanted to be able to plug IoT devices into network switches and collect information about where they were plugged in and then call home. There are some very good mechanisms for this is networking. The best, which is a standard is LLDP or the Link Layer Discovery Protocol. The problem with this being a standard is that it came too late and most network devices in operation don't support it. At least not out of the box they don't. Each vendor has their own protocol for accomplishing this though. Cisco for example uses CDP or the Cisco Discovery Protocol. This is a closed protocol which is only mostly understood. Wireshark, the ultimate network sniffer doesn't properly parse CDP packets... at least it doesn't name the fields appropriately.

### The first challenge

CDP is or was documented at some point by Cisco. That documentation has all but disappeared in the constant mish-mash of Cisco.com where links go dead so often that not even their own quality assurance can keep up with it. So, I was at the mercy of using Wireshark and also using other open source implementations of CDP as reference. The good news is, that now, I believe I have it all sorted out... at least the parts of it I consider interesting.

### The second challenge

SNAP! Well I mean the [Subnetwork Access Protocol](https://en.wikipedia.org/wiki/Subnetwork_Access_Protocol). This is a fairly archaic method of providing a common frame encapsulation format on as many forms of media as possible. This basically allows the concept of an "EtherType" to be present on anything from Ethernet to PPP connections via modem.

#### 802.2 vs. 802.3

Almost all modern networking today uses 802.3 frame formatting for Ethernet. This standard goes back a REALLY REALLY long time. The real problem with this is that 802.2 hasn't really been used commonly since the 1980s. It was used for things like [IPX networking from Novell](https://en.wikipedia.org/wiki/Internetwork_Packet_Exchange) and [AppleTalk from Apple](https://en.wikipedia.org/wiki/AppleTalk) and [NetBEUI from Microsoft](https://en.wikipedia.org/wiki/NetBIOS). But with TCP/IP, when communicating over Ethernet, the preferred method is 802.3.

The challenge of this arises in that it is currently impossible to write code for [Windows 10 IoT Core](https://developer.microsoft.com/en-us/windows/iot) that can access SNAP frames. This is because there's no filter API available in the Windows 10 Universal driver API at this time.

It's also extremely difficult to do for Linux since Alan Cox never really fully implemented SNAP beyond what was required to support AppleTalk. The 802.2 modules and the PSNAP module does implement footprint required to send and receive SNAP frames, but it was never fully tied into the rest of the code. This means that from user mode, the only way to access the CDP frames is to write code which operates on raw frames. This means you lose all the advantages of the different drivers that are written for your code.

As I hope to run my CDP client over PPP and DMVPN, I did not want to use raw sockets to perform CDP operations to get access to the SNAP frames and to be able to transmit them. I'd have to rewrite major portions of the kernel to create headers for all the different layer-1 and layer-2 interfaces I'd be operating over.

This meant I'd have to do some kernel coding.

### The third challenge.

I don't like doing kernel coding.

No really. The Linux Kernel while a thing of beauty for many reasons is a mountain of spaghetti. I mean, finding anything in the kernel tree is like looking for a needle in a stack of needles. We're talking millions of lines of code written by countless people of varying differing skill levels and often based on unrealistic deadlines. The naming conventions alone should be enough to scare away any developer with the slightest self-respect.

#### The GP-HELL

Ok, so, the GPL is one of the most complex licenses in the world. It basically says that as long as your willing to make all your code GPL, then you can code with other GPL code. As the Linux Kernel is GPL, this means that if you want to make parts of your code non-restrictive (BSD), then you can have problems.

#### The C programming language

Say what you will about C, but it's not my favorite language. I moved on to greener pastures a long time ago.

To do the job properly, I needed to write a library to handle all the non-kernel specific code. There are multiple reasons for this, the most important of course is that I wanted unit testing in place. While there are some unit test frameworks available for testing kernel modules, they are not nearly as advanced as others. In addition, since I'd be working with dynamic memory allocation, I wanted to be able to use Valgrind. I recently found [KEDR](https://github.com/euspectre/kedr/wiki) which is similar to Valgrind which I hope to use as well.

When writing a library in C and facing restrictions like the Linux Kernel GPL, it is very difficult to make BSD friendly code that uses any of the systems within the Linux Kernel. And to be frank, this is 2018 and C still doesn't have the concept of a string nor an internal definition of a list. This means that static code analysis is going to be highly limited at best. There are advantages of C, but it's very difficult to appreciate them when you're handling all your error checking with return codes, manually managing buffers for strings and reinventing the wheel for collections.

To compensate for some of what I mentioned before, C almost forces you to program using a preprocessor. This makes code a mangled mess and I've actually received hundreds of lines of meaningless error messages from GCC because of a typo in the name of a variable passed to the list functions in the kernel. This list functions were actually nasty hacks written as C preprocessor macros to impersonate the behavior generics. Given that almost all C projects today make heavy use of generics in this manor, isn't it about time someone proposes a language extension for generics instead of programming in a "non-language"?

Let me also say as a last major whining fit that I don't like that C has no concept of what a project is. As such each file and function and structure in C is all alone and as such requires things like forward declaration and other such nasties. I simply just don't like coding in it.

### The worst challenge

The programmer is the worst challenge.

I'm an obsessive compulsive developer. I believe that I'm a really good programmer who is extremely disciplined at doing things consistently if not cleanly. And I try to be clean too. But to be honest, I am at a total loss when I am up against coding a kernel module in C. I wrote over 8000 lines of code to do what should have been possible in under 1000 in more or less any modern language. This cost me a lot of time, but the job is done and I genuinely believe I'll be able to read and understand the code later on.

Even now, I almost have heart failure thinking that I might have misunderstood the kernel's buffer handling and I fear there's a memory leak. I WILL FIND IT!

## The plan

### Good tools!

So, the first step is to write as much of the code as possible using good editors, good debuggers and good everything. Let's look at my toolkit that I ended up with by the time I was finished.

* Windows Subsystem for Linux

OH MY GOODNESS!!! I love you Microsoft!!! I mean really, thanks to WSL, I want to hug my computer. It's like having Linux with an actual working GUI. It is an absolutely blessing! I get almost all my beloved Ubuntu goodies with a responsive user interface that just works. It's like Christmas every time I use it.

The main problem when writing a Linux kernel module on WSL is that WSL doesn't actually run the Linux Kernel. It's a subsystem which simulates the system call interface as a compatibility layer. There are some great articles out there on how it works, and it's really amazing stuff. The subsystem isn't just an API translation, but it's also almost like a container with awesome security. The primary problem with it is that most antivirus software (including Microsoft's) is bloody confused by it and it causes the anti-malware software to spin like crazy on all reads and writes.

* Microsoft Visual Studio 2017

VS2017 has C++ project types for writing Linux code these days. What you do is point it to a Linux system that can be reached via SSH and it more or less does the rest. Conveniently, you can configure WSL as an SSH server as you always have on any other Linux since... well it's Linux without the Linux kernel and that's all.

* VMware Workstation

Well, I was trying to build a debug kernel for Ubuntu on Hyper-V... and I was sure that the reason I couldn't was because I was in Hyper-V and that I couldn't get past SecureBoot. Well it turns out I had the same problems on VMware, so I'm pretty sure the real problem is my own lack of knowledge.

I'm going to remove VMware and switch back to Hyper-V and also experiment with using Alpine Linux and QEMU instead. I didn't realize it, but Microsoft's actually done a lot of work to get [accelerated QEMU working side-by-side with Hyper-V](https://lists.nongnu.org/archive/html/qemu-devel/2018-01/msg02742.html) by implementing support for the new [Windows Hypervisor Platform](https://docs.microsoft.com/en-us/virtualization/api/). So, there's simply no reason to waste time using VMware anymore.

* VisualKernel and VisualGDB

I've now spent way too much time working on these tools and not getting anywhere with them. They seemed great to begin with, but to be fair, I think I have enough to work with now that there's no need to compile a Linux kernel anymore and using GDB from the WSL prompt is more than good enough. I'm pretty sure that I can get Visual Studio Code to remote debug the Linux Kernel graphically.

VisualKernel is a pretty cool tool, but it simply didn't add enough value to justify using it. In fact, I believe it actually simply slowed me down.

* Visual Studio Code

So, for all the code I couldn't do in Visual Studio itself, I used Visual Studio Code running on Windows. Thanks to the built in support for WSL and bash, it was possible to keep a terminal window open and with a few lines of script, I had a functional build environment running. Just tar the source, send it to the virtual machine, untar them and build. I know I could have used a shared folder, but it really didn't add much time to the build task, so it wasn't worth having to keep the VM running all the time to have access to the code.

### Good people!

I hopped onto the IRC channel for Kernel development which is of course appropriately named #kernelnewbies.. yeh.. that took a while to find.

The people there were very knowledgeable and I think that since people have grown up a lot since I last worked on Linux Kernel code, it wasn't the standard "Hahaha you don't know how to do that!!!" and "You dumbass, you why would you want to do that!?!?!" responses which made me tired of the Linux world many years ago. The answers I received instead were accurate and helpful and I was very happy the people in the channel provided me support even if they didn't always agree with how I was trying to accomplish my task.

## Conclusion of Part 1

Well, I've written enough for now. I'll post a follow up entry after I spend the day with my family. It is Sunday and I've been at work since 4am and it's now 1:35pm. I think this is a sign that it's time to consider food or possibly a shower at least.

In the next article, I'll write about the design and the division of labor between the library and the module code itself.
