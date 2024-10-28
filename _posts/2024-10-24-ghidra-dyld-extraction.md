---
layout: post
title: Viewing MacOS system libraries in Ghidra
date: 2024-10-24
tags: macos reversing
---

# Intro
I recently cracked open a copy of [*OS Internals Volume 1](https://www.amazon.com/MacOS-iOS-Internals-User-Mode/dp/099105556X) and wanted to follow along with some of the system library examples show in the book. I'm a huge fan of [Ghidra](https://ghidra-sre.org/) for my reverse engineering, so I fired it up, created a new project, and navigated to `/usr/lib` to find... no `libSystem.B.dyld` present. "How annoying" I thought to myself, "they must have moved the system libraries since the book came out". The book was published in 2019, and MacOS has a storied history of moving things around, so this felt like the most reasonable explanation. Striving to be the self-sufficient reverse engineer, I ran the following `otool` command to find the system library:

```
$ otool -L /bin/ls
/bin/ls:
	/usr/lib/libutil.dylib (compatibility version 1.0.0, current version 1.0.0)
	/usr/lib/libncurses.5.4.dylib (compatibility version 5.4.0, current version 5.4.0)
	/usr/lib/libSystem.B.dylib (compatibility version 1.0.0, current version 1345.120.2)
```

Um... What?

# Dynamic Linker Shared ~~Cache~~

After a lot of searching around, I finally found [this](https://developer.apple.com/forums/thread/655588?answerId=665804022#665804022) developer forum answer from Apple. 

> Given the above reality, the libraries in /usr/lib are no longer needed:
>    - Developer tools should be looking at the stub libraries in the appropriate SDK.
>    - The runtime doesn’t use these libraries because they’ve been rolled into the dynamic linker shared cache [4].
>
> And that’s why they were removed.

As all great nuggets of wisdom are, this one was tucked inside the footnote of the post (emphasis is mine):

> [4] This term is slightly off. Historically this was actually a cache that the system would build from its working set of libraries and frameworks. On macOS 11 this is no longer a cache because **the original files are no longer present on the system. Rather, this shared cache is authored by Apple and distributed in toto via the software update mechanism.**

This was also confirmed with the patch notes for Big Sur:

> New in macOS Big Sur 11 beta, the system ships with a built-in dynamic linker cache of all system-provided libraries. As part of this change, copies of dynamic libraries are no longer present on the filesystem. Code that attempts to check for dynamic library presence by looking for a file at a path or enumerating a directory will fail. Instead, check for library presence by attempting to dlopen() the path, which will correctly check for the library in the cache.

Instead of having the binaries themselves on disk, Apple has opted to package all of them into a "cache" file that will be immediately loaded into memory on startup. Since Apple controls the dynamic loader, it's able to intercept any `dlopen` or other attempts to link against the library and feed the cached version instead. This explains why `otool` thinks we're getting our libraries from that file location, even though they aren't there. Reading through others' thoughts on this, it seems to have some kind of security/performance implications, but I am always very suspicious of when those two things get waved around randomly.[^1]

# Ghidra To The Rescue

What it does mean is that we can't just import these into Ghidra like normal libraries, so reading through them is going to be more difficult. Or so I thought, but it turns out Ghidra already built the functionality to read these "cache" files, since they really just end up being binary archives. We can access this functionality by going to this menu:

![File > Open File System](/assets/images/open-file-system.png)

Then, navigate to the location of the cache file (in macOS 14.7, it lives at `/System/Cryptexes/OS/System/Library/dyld/dyld_shared_cache_arm64e` and maybe `/System/Library/dyld/dyld_shared_cache_arm64e`) and click it. You should be greeted with the following window:

![](/assets/images/open-dylib-cache-ghidra.png)

From here, you can select which libraries you want to import, and if you import with the "Load Libraries From Disk" option, then you'll automatically pull in all the other system dependencies you need to link against that. Remember to import as the format "Mac OS X Mach-O" if you want to get the automatic dynamic library chain import functionality.

# For Things Outside Ghidra

[I'm told](https://mjtsai.com/blog/2020/07/27/hopper-for-apple-silicon-and-big-sur/) if you are using Hopper, then they can also do all of this for you. Hopper has a pretty good track record of doing *OS stuff correctly, so I would believe this claim, but I haven't tested it yet.

For everyone else, I recommend [dyld-shared-cache-extractor](https://github.com/keith/dyld-shared-cache-extractor). This project calls into `dyld`'s own methods as a way to extract all the libraries out to a folder in the filesystem. The setup is pretty basic, but it will get you all the files back on system, to be reversed however you may please.

Happy hacking!

[^1]: For a few really good reasons (that I definitely learned from) about why this might be better for performance specifically, see [this comment from lgerbarg on Lobste.rs](https://lobste.rs/s/oxdvms/viewing_macos_system_libraries_ghidra#c_4pplcz). 