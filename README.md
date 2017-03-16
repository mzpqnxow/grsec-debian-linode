# grsec-debian-linode
Quick guide to getting a grsec kernel running on a Debian Linode VM

## Audience

This guide is as much for my own documentation as it is for anyone else. I hope someone finds it useful despite it not being a terribly difficult process. I rarely build custom kernels so this is a huge timesaver for me when I need it

## Credits

This is based on various posts on The Internet (TM) and excellent documentation from Debian and to a lesser extent, Linode. Also, much help from the oh so kind, humble and modest Bradley Spengler. His family is also pretty nice. Really nice.

## Steps

### Create and boot your Linode

You will want to choose Debian 8 and (important) for the profile you will want to choose the Linode kernel that is closest to the kernel your version of grsec uses. This is to make the kernel configuration step much easier. It leaves out a lot of unnecessary cruft by default, which can be time consuming to disable via make oldconfig && make menuconfig

Once you have booted, make sure your uname -a output roughly matches your grsecurity patch version output. In my case, uname -a revealed 4.4.5 while grsecurity was 4.4.54. Close enough.

### Install some prerequisites

These should cover it, though I may have missed 1 or 2:

```
apt-get install gcc-4.9-plugin-dev build-essential bison flex kernel-package libncurses5 paxctl
```

### Download the Linux kernel source and grsecurity patch and extract / patch

For this example, I will be using the following:

From https://grsecurity.net/download.php#test: grsecurity-3.1-4.4.54-201703150048.patch or the test patch if you don't pay the ransom Bradley demands due to Linus being rude to him
From kernel.org: linux-4.4.54.tar.xz

Don't forget to get the GPG signatures for each file and test their validity

```
# cd /usr/src
# wget grsecurity-3.1-4.4.54-201703150048.patch
# wget linux-4.4.54.tar.xz
# tar -xvf linux-4.4.54.tar.xz
# cd linux-4.4.54
# patch -p1 < ../grsecurity-3.1-4.4.54-201703150048.patch
# zcat /proc/config.gz > .config
# make oldconfig
# make menuconfig
```

### Configuring grsecurity

In the make menuconfig option, grsecurity is found under "Security". Enable it and choose your features starting with the automatic 'desktop' or 'server' build

From there, you can choose to enable/disable specific features. I recommend leaving RBAC enabled for servers, it's a very powerful and mature feature (>10 years old) and has a very advanced learning mode that supports reduction and all sorts of other cool things that produce a very close to perfect policy. It will, however, require quite a bit of configuring, especially if you are going to properly use role transitions to protect the kernel from the root user.

Some good features for multi-user machines where you would like to limit mischief, but don't want to use the full RBAC, I suggest making use of the Trusted Path Execution (TPE) feature, which restricts non-root users to running only root owned read-only binaries. Be sure to use a chmod scheme on the more powerful interpreters. For example:

```
# for INTERP in /usr/bin/python* /usr/bin/perl* /usr/bin/awk* /usr/bin/ruby* /usr/bin/java*
do
  chown root.trustexec ${INTERP}
  chmod 750 ${INTERP}
done
```

In this example case, make sure the users with TPE restrictions are NOT in the group 'trustexec' and that users you do trust are in this group. The GID valueis configurable in make menuconfig.

You can do similar things to restrict client and server sockets per GID, if you'd like to restrict users from establishing sockets, internally or externally. Optionally, you can use iptables egress rules, but those won't help reduce your UNIX socket attack surface locally. Again, the correct way here is to use RBAC, but that's more work. Security is hard.

Keep in mind this is only as effective as you are thorough. The correct way to do this is by whitelisting things granularly using the RBAC system. That said, if you are reasonably good at attacking systems, you should be able to disable all of the more powerful interpreters on the system

### Building the Debian package

At this point, you can dpkg --list | grep -E '(linux-header|linux-image)' and use dpkg --remove on each, as well as dpkg --purge. If things go wrong, Linode will allow you to fix your kernel via the web UI later

Ok, build the packages now

```
# rm -f /usr/src/linux;ln -sf $PWD /usr/src/linux
# make-kpkg --rootcmd fakeroot kernel_image kernel_headers --initrd --revision=2
```

You should now have two Debian packages in /usr/src. You can install them with dpkg now.

```
# dpkg -i /usr/src/linux-headers-4.4.54-grsec_1_amd64.deb /usr/src/linux-image-4.4.54-grsec_1_amd64.deb
# update-initramfs -c -k 4.4.54
# update-grub2
```

You're all set now, if you like you can check /boot/grub/grub.cfg and make sure there is a 'grsec' string in the file. You can also look in /boot and see the kernel and initrd file clearly labelled:

```
# ls -l /boot
-rw-r--r-- 1 root root   72967 Mar 16 03:13 config-4.4.54-grsec
drwxr-xr-x 5 root root    4096 Mar 16 03:55 grub
-rw-r--r-- 1 root root 2811931 Mar 16 02:20 initrd.img-4.4.54
-rw-r--r-- 1 root root 2819146 Mar 16 03:55 initrd.img-4.4.54-grsec
-rw------- 1 root root 2745548 Mar 16 03:37 System.map-4.4.54-grsec
-rw------- 1 root root 5360672 Mar 16 03:37 vmlinuz-4.4.54-grsec
```

Remember /boot should be mode 700, it should be by default but it can't hurt to check

### Update your image profile via the Linode web UI

In your profile, click 'Edit'. You will want to change the Boot Settings setting named 'Kernel' to GRUB2. If things go wrong, you can change this back to a Linode kernel and things will boot fine as before, so don't worry about breaking anything. Save the change.

### Reboot and await success, or perhaps failure

You can now reboot from software or via the web UI. You should be logged into the console as it reboots so you can identify any errors. In my case, the following entries were necessary in my /etc/default/grub:

```
# cat /etc/default/grub | grep -v ^#
$ cat /etc/default/grub | grep -v ^# | uniq

GRUB_DEFAULT=1
GRUB_TIMEOUT=30
GRUB_DISTRIBUTOR=`lsb_release -i -s 2> /dev/null || echo Debian`
GRUB_CMDLINE_LINUX_DEFAULT=""
GRUB_CMDLINE_LINUX="console=ttyS0,19200n8"
GRUB_SERIAL_COMMAND="serial --speed=19200 --unit=0 --word=8 --parity=no --stop=1"

GRUB_TERMINAL=serial

GRUB_DISABLE_LINUX_UUID=true

GRUB_DISABLE_OS_PROBER=true
GRUB_GFXPAYLOAD_LINUX=text
```

That's it. Unless you fiddled too much in menuconfig and broke something by disabling KVM paravirtualization or something, it should *just work* even with high security + server autoconfiguration enabled, along with the RBAC system. From here, if you want to strip down the kernel further once it is working, build with a revision of 2 and repeat the above steps. Make sure you do a make-kpkg clean or a make distclean in /usr/src/linux

### Using the RBAC system

If you don't want to actually use the RBAC system, I still recommend you put it in learning mode for a week or two and let it produce a policy just so you can see how granular it is and well the learning mode works. Follow these steps

#### Install gradm from https://grsecurity.net/download.php?#test

Visit https://grsecurity.net/download.php#test and find the gradm tarball. In this case, I will use gradm-3.1-201701031918.tar.gz

```
$ wget gradm-3.1-201701031918.tar.gz
$ tar -xvzf gradm-3.1-201701031918.tar.gz
$ cd gradm-3.1
$ make -j && sudo make install
```

To enable learning mode (minimal performance impact, will not enforce any permissions)

```
# gradm -F -L /etc/grsec/learning.logs
```

You will be prompted for some role passwords, depending on how you plan to use (or not use) the RBAC system, these should be carefully chosen and not forgotten.

### Producing a policy from learning logs after 1-2 weeks of system usage

Convert the learning logs into a policy

```
# gradm -F -L /etc/grsec/learning.logs -O /etc/grsec/policy.output
```

You might be prompted to turn learning mode off using gradm -D first, which will require the admin role password.

Once this is done, peruse /etc/grsec/policy.output and marvel at how granular some of the permissions are, especially if you have a system with a dedicated role (i.e. a web server only, or a mail server, etc)

It is also very cool for multi-user systems where users only check e-mail or use some basic functionality- when you don't want users snooping around the box. With the right policy, users won't even be able to see / let alone execute /bin/cat or whichever other applications a user might run without necessity.

## Test and consider using paxctld

I am not familiar with paxctld, I have always use paxctl manually to opt certain executables out of certain PAX mitigations whenever necessary, but you might want to look into paxctld as I believe it automates this via some sort of profile/configuration system.

It is available at https://grsecurity.net/download.php#test

## Test userspace memory protections using paxtest

You can also now try paxtest to make sure userspace memory protections are working as expected

```
$ sudo apt-get install paxtest
$ ./paxtest blackhat
PaXtest - Copyright(c) 2003,2004 by Peter Busser <peter@adamantix.org>
Released under the GNU Public Licence version 2 or later

Writing output to /home/a/paxtest.log
It may take a while for the tests to complete
Test results:
PaXtest - Copyright(c) 2003,2004 by Peter Busser <peter@adamantix.org>
Released under the GNU Public Licence version 2 or later

Mode: Blackhat
Linux gutter 4.4.54-grsec #1 SMP Thu Mar 16 03:36:35 UTC 2017 x86_64 GNU/Linux

Executable anonymous mapping             : Killed
Executable bss                           : Killed
Executable data                          : Killed
Executable heap                          : Killed
Executable stack                         : Killed
Executable shared library bss            : Killed
Executable shared library data           : Killed
Executable anonymous mapping (mprotect)  : Killed
Executable bss (mprotect)                : Killed
Executable data (mprotect)               : Killed
Executable heap (mprotect)               : Killed
Executable stack (mprotect)              : Killed
Executable shared library bss (mprotect) : Killed
Executable shared library data (mprotect): Killed
Writable text segments                   : Killed
Anonymous mapping randomisation test     : 28 bits (guessed)
Heap randomisation test (ET_EXEC)        : 23 bits (guessed)
Heap randomisation test (PIE)            : 35 bits (guessed)
Main executable randomisation (ET_EXEC)  : 28 bits (guessed)
Main executable randomisation (PIE)      : 28 bits (guessed)
Shared library randomisation test        : 28 bits (guessed)
Stack randomisation test (SEGMEXEC)      : 35 bits (guessed)
Stack randomisation test (PAGEEXEC)      : 35 bits (guessed)
Arg/env randomisation test (SEGMEXEC)    : 39 bits (guessed)
Arg/env randomisation test (PAGEEXEC)    : 39 bits (guessed)
Randomization under memory exhaustion @~0: 28 bits (guessed)
Randomization under memory exhaustion @0 : 28 bits (guessed)
Return to function (strcpy)              : paxtest: return address contains a NULL byte.
Return to function (memcpy)              : Killed
Return to function (strcpy, PIE)         : paxtest: return address contains a NULL byte.
Return to function (memcpy, PIE)         : Killed
$
```

This is a good result. Good luck with Gera's Insecure Programming challenges now :>

One weakness you will have unless you bootstrapped your entire system using PIE- the main executable text section will not be randomized. This is usually not a major problem given all of the other mitigations in place and the relatively small size of this section, making it not so rich of an area for LOP (libc oriented programming)

If you want to build a PIE distribution, your best bet might be Gentoo, but I haven't done this for many years so there may be some other distribution that let's you do this easily. Try to Google for "full pie linux system" or something like this

## Enjoy a significantly hardened system

With or without RBAC, you now have a system that is significantly hardened against both userspace memory corruption attacks and logic/race condition vulnerabilities as well as a kernel that is hardened against many classes of attacks, which means both public and private bugs will be squashed. I say most because the perfect bugs will still be exploitable, though they will often be more difficulty since grsecurity hides all of the systems symbols and there is no RPM somewhere on the Internet or on your local machine with kernel symbol addresses. The types of bugs that will kill you are those with read/write anywhere, especially with read relative primitives. This means you've reduced your susceptibility to 0day by a non-negligible quanity.
