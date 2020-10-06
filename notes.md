# Notes from Linux From Scratch

This document contains notes to myself, somewhat stream of consciousness style,
from when I went through Linux From Scratch 9.1, non-systemd version.

http://www.linuxfromscratch.org/lfs/download.html
http://www.linuxfromscratch.org/lfs/downloads/9.1/
http://www.linuxfromscratch.org/lfs/downloads/9.1/LFS-BOOK-9.1.pdf

Headers refer to the section of the book I took the notes in.


# 2.1 Introduction

Host system is Dell XPS 13 9370 running Arch GNU/Linux x86_64 updated as of
2020-08-31.

```console
$ uname -a
Linux xeps 5.8.5-arch1-1 #1 SMP PREEMPT Thu, 27 Aug 2020 18:53:02 +0000 x86_64 GNU/Linux
```

Target system is a Thinkpad X200.


# 2.2 Host System Requirements

`pacman -Q` can be used to list installed versions of everything for checking
the version requirements.


# 2.4 Creating a New Partition

X200 ssd removed and connected to host machine via a USB to SATA cord. I'll use
an existing empty partition on this drive that I had saved for trying out other
distributions.

- LFS install target `/dev/sda3`
- Existing swap `/dev/sda7`

# 2.4.1.4 Convenience Partitions

I am just going to use one partition to keep things simple, `/boot`, `/home`
etc. will all be on `/dev/sda3`.

# 2.5 Creating a File System on the Partition

Using `ext4`.

# 2.6 Setting The $LFS Variable

Using `/storage/mnt/lfs` since I have most of the disk space on this box
mounted at `/storage`.

# 3.1 Introduction

`wget-list` and `md5sums` are files available from
http://www.linuxfromscratch.org/lfs/downloads/9.1/.

# 3.3 Needed Patches

`wget-list` includes the patches to download so I don't need to do anything
here.

# 4.5 About SBUs

Use all the cores.

```console
$ nproc
8
$ export MAKEFLAGS='-j8'
```

# 5.1 Introduction

Two steps to build:

1. Build host-independent toolchain: compiler, assembler, linker, libraries, and
   a few utilities
2. Use this toolchain to build the other essential tools.


# 5.2 Toolchain Technical Notes

A platform's dynamic linker (also called dynamic loader, but not to be confused
with the standard linker `ld` that is a part of Binutils) is provided by Glibc.
`ld-linux.so.2` for 32-bit and `ld-linux-x86-64.so.2` for 64-bit. To identify
the dynamic linker a system is using `readelf -l <name of binary> | grep
interpreter`.

```console
$ readelf -l $(which ls ) | grep interpreter
      [Requesting program interpreter: /lib64/ld-linux-x86-64.so.2]
```

The "target triplet" of my host is `x86_64-pc-linux-gnu`.

First build of Binutils and GCC produce a cross-linker and cross-compiler to
ensure a completely clean build. This is done by changing the "vendor" field
target triplet by using the LFS_TGT variable. The temporary libraries are
cross-compiled to ensure no headers or libraries from the host are used. The
GCC source is manipulated (:eyebrow raise:) to tell the compiler which target
dynamic linker to use.

Binutils is installed first because `configure` runs tests for GCC and Glibc,
this will help ensure these are configured correctly. Otherwise problems might
not arise until the end of building all the things.

`ld --verbose | grep SEARCH` shows linker library search locations in order.

The assembler and linker used for the GCC build should be in
`/tools/$LFS_TGT/bin`, PATH is not used.

To see what standard linker `gcc` itself will use though run `gcc
-print-prog-name=ld`. Passing `-v` while compiling a dummy program will show
more details about preprocessor, compilation, and assembly stages, including
`gcc`'s included search paths and their order.

The Linux API headers are built to allow the standard C library to interface
with features from the Linux kernel.

Then Glibc itself is built.

On the second build of the Binutils, we can use `--with-lib-path` to control
`ld`'s library search path.

On the second build of GCC it's sources are modified to use the new dynamic
linker that was built. Not doing this would link the dynamic linker from the
host system's `/lib` directory embedded into them. After this the core
toolchain is self-contained and self-hosted.

Then the chroot can be entered and Glibc can be built again.

Then everything else will be built against the Glibc in `/tools`.


So overall something like this I think:

1. Binutils
2. GCC
3. Linux API headers
4. Glibc
5. Binutils
6. GCC

Install a bunch of other temporary tools I guess?

7. in chroot - Glibc


# 5.4.1 Installation of Cross Binutils

Put all the commands in `build.sh`:

```shell
#!/bin/sh

set -ex

../configure --prefix=/tools \
             --with-systeroot=$LFS \
             --with-lib-path=/tools/lib \
             --target=$LFS_TGT \
             --disable-nls \
             --disable-werror

make

case $(uname -m) in
  x86_64) mkdir -v /tools/lib && ln -sv lib /tools/lib64 ;;
esac

make install

echo "$DONE!"
```

Execute it wrapped with `time` to see how long it takes to for the Standard
Build Unit (amount of time it takes to build Binutils) on my system.

```
time ./build.sh
```

37 seconds.

# 5.5.1 Installaction of Cross GCC

`build.sh`:

```shell
#!/bin/sh

set -ex

../configure --prefix=/tools                                \
             --target=$LFS_TGT                              \
             --with-glibc-version=2.11                      \
             --with-sysroot=$LFS                            \
             --with-newlib                                  \
             --without-headers                              \
             --with-local-prefix=/tools                     \
             --with-native-system-header-dir=/tools/include \
             --disable-nls                                  \
             --disable-shared                               \
             --disable-multilib                             \
             --disable-decimal-float                        \
             --disable-threads                              \
             --disable-libatomic                            \
             --disable-libgomp                              \
             --disable-libquadmath                          \
             --disable-libssp                               \
             --disable-libvtv                               \
             --disable-libstdcxx                            \
             --enable-languages=c,c++

make

make install

echo "DONE!"
```

`time ./build.sh`

6 minutes 13 seconds.

# 5.6.1 Installation of Linux API Headers

Built headers go into `/tools/include`.

# 5.7.1 Installation of Glibc

`build.sh`:

```shell
#!/bin/sh

set -ex

../configure --prefix=/tools                    \
             --host=$LFS_TGT                    \
             --build=$(../scripts/config.guess) \
             --enable-kernel=3.2                \
             --with-headers=/tools/include

make

make install

echo "DONE!"
```

`time ./build.sh`

3 minutes 16 seconds.

Testing dummy build:

```console
$ cd $LFS/sources
$ echo 'int main(){}' > dummy.c
$ $LFS_TGT-gcc dummy.c
$ readelf -l a.out | grep ': /tools'
      [Requesting program interpreter: /tools/lib64/ld-linux-x86-64.so.2]
```

Looks good, it is using files from within the `/tools` directly which is
symlinked to `$LFS/tools`.

# 5.10.1 Installation of GCC

8 minutes 39 seconds.

# 5.11.1 Installation of Tcl

`tclsh8.6` did not exist in the build directory that was supposed to be
symlinked, but `ln`'s confusing syntax links it from `/tools/bin/tclsh8.6 ->
/tools/bin/tclsh`. So it's all good.

# 5.* Install a bunch of stuff for the temporary toolchain

Variations on `./configure --prefix=/tools ... && make && make install` over
and over.

# 5.35 Stripping

Opted to strip the files. Not deleting documentation. Removing unneeded files.

21GB available, should be plenty. Though it is probably possible to create the
temporary toolchain on a normal directory of the host, not sure why it done on
a completely separate partition?

# 5.36 Changing Ownership

Backing up all these created tools via:

```console
$ mkdir -p /storage/lfs/9.1
$ cp -r $LFS/tools /storage/lfs/9.1/tools
```

# 6.4 Entering the Chroot Environment

Okay now actually entering the chroot.

# 6.19.1 Installation of GMP

Since the host machine is way more powerful than the target machine, I am
using the generic GMP libraries as mentioned in the second note.

```console
(lfs chroot) root:/sources/gmp-6.2.0# cp -v configfsf.guess config.guess
'configfsf.guess' -> 'config.guess'
(lfs chroot) root:/sources/gmp-6.2.0# cp -v configfsf.sub config.sub
'configfsf.sub' -> 'config.sub'
```

# 6.25.1 Installation of GCC

The build and tests took 177 minutes 46 seconds.


# 6.40 Inetutils-19.4

My goodness this is a gauntlet of installing things.

I notice this installs things like `ifconfig`, it seems Arch linux uses the
`iproute2` package instead (which provides the `ip` command).

It isn't clear to me how easy this could be switched out, I am going to stick
to the book to make sure I can get a functioning system before trying anything
fancy.

# 6.5.1 Python-3.8.1

I realize now I forgot to investigate the note 2 installs back for Libffi-3.3
to specify CFLAGS and CXXFLAGS to create a more generic build, rather than one
with optimizations for the current processor. So I'm backpedaling a bit here. I
hope the openssl install won't get borked.

# 6.49.1 Installation of Libffi

Okay what CFLAGS and CXXFLAGS options should be used?

Gentoo wiki has some info
https://wiki.gentoo.org/wiki/GCC_optimization

> If the type of CPU is undetermined, or if the user does not know what setting
> to choose, it is possible use the -march=native setting. When this flag is
> used, GCC will attempt to detect the processor and automatically set
> appropriate flags for it. However, this should not be used when intending to
> compile packages for different CPUs!

But it doesn't say what to use or where to look for it.

Okay looking at the GCC docs I think I'll want `-march=x86-64`, which is
referred to in the LFS note for the `--with-gcc-arch` option.
https://gcc.gnu.org/onlinedocs/gcc/x86-Options.html

```console
$ CFLAGS="-march=x86-64"
$ export CFLAGS
$ CXXFLAGS="-march=x86-64"
$ export CXXFLAGS
```

Then do the `configure` with a modified `--with-gcc-arch` option:

```console
$ ./configure --prefix=/usr --disable-static --with-gcc-arch=x86-64
```

Then go through the normal install instructions, then:

```console
$ unset CFLAGS
$ unset CXXFLAGS
```

Seems to have completed successfully. Back to following the install order
(redoing OpenSSL and what I had done for Python).

# 6.52.1 Installation of Ninja

Not using NINJAJOBS, trying my luck at the default behavior.

# 6.54.1 Installation of Coreutils

Getting an error while doing `make`.

```
  CCLD     src/expand
/usr/bin/ld: src/expand.o: in function `expand':
expand.c:(.text+0x422): undefined reference to `mbfile_multi_getc'
collect2: error: ld returned 1 exit status
make[2]: *** [Makefile:8824: src/expand] Error 1
make[2]: Leaving directory '/sources/coreutils-8.31'
make[1]: *** [Makefile:12651: all-recursive] Error 1
make[1]: Leaving directory '/sources/coreutils-8.31'
make: *** [Makefile:6831: all] Error 2
```

Seems this person encountered it as well but there is no resolution
https://lfs-dev.linuxfromscratch.narkive.com/SmFdIStZ/chapter-6-coreutils-8-28.

Also https://github.com/dslm4515/Musl-LFS/issues/11

Also this thread
https://www.mail-archive.com/lfs-support@lists.linuxfromscratch.org/msg06363.html

Tried to get more debug information with `make V=1 -j1 > log.txt 2>&1`

And now it was successful, probably the `-j1`?

Continuing on.

# 6.64 IPRoute2-5.5.0

Ah iproute2 is installed in addition to inetutils, interesting.

# 6.67 Make-4.3

makeception

# 6.70 Tar-1.32

tarception

# 6.71.1 Installation of Texinfo

If the info pages ever get out of sync

```shell
pushd /usr/share/info
rm -v dir
for f in *
  do install-info $f dir 2>/dev/null
done
popd
```

# 6.73.1 Installation of Procps-ng

Getting some test failures that don't seem to be mentioned in the docs.

```
Running ./ps.test/ps_output.exp ...
FAIL: ps with output flag bsdstart,start,lstart
FAIL: ps with output flag bsdtime,cputime,etime,etimes
Running ./ps.test/ps_personality.exp ...
Running ./ps.test/ps_sched_batch.exp ...

                === ps Summary ===

# of expected passes            8
# of unexpected failures        2
```

Similar errors seen here
http://lists.linuxfromscratch.org/pipermail/lfs-support/2019-March/052668.html
And the suggestion was to continue on...
Continuing on.

# 6.74.2 Installation of Util-linux

I made a copy-paste error that left the `./configure` off of a command. I'm a
bit paranoid perhaps I made that mistake previously (though I don't the
make/make install's would work at all if the ./configure had not been run).
Trying to check the bash history `/root/.bash_history` only contains up to the
invocation of the rebuilt `bash`. Running `history >
/root/new-bash-history.txt` provides the actual command history for the current
shell. Looking through the history I don't think I made this mistake anywhere
else.


Random note: I'm running all the tests for things in this chrooted portion,
want to make sure these things are working.

Also I'm installing all documentation mentioned in the book.

# 6.78.2 Configuring Eudev

> `udevadm-hwdb --update`
> This command needs to be run each time the hardware information is updated.

I guess I'll need to do this when the target system boots up then, since it'll
be on different hardware?

# 6.80 Stripping Again

I am *not* going to remove the debug symbols.

# 6.81 Cleaning up

Double checking the virtual kernel file systems are mounted

```console
(lfs chroot) root:/# mount
/dev/sda3 on / type ext4 (rw,relatime)
dev on /dev type devtmpfs (rw,nosuid,relatime,size=8036104k,nr_inodes=2009026,mode=755)
devpts on /dev/pts type devpts (rw,relatime,gid=5,mode=620,ptmxmode=000)
proc on /proc type proc (rw,relatime)
sysfs on /sys type sysfs (rw,relatime)
tmpfs on /run type tmpfs (rw,relatime)
```

Looks good to me.


I *am* going to opt to remove the static libraries that were only used for
regression testing.

```shell
rm -f /usr/lib/lib{bfd,opcodes}.a
rm -f /usr/lib/libbz2.a
rm -f /usr/lib/lib{com_err,e2p,ext2fs,ss}.a
rm -f /usr/lib/libltdl.a
rm -f /usr/lib/libfl.a
rm -f /usr/lib/libz.a
```

# Chapter 7. System Configuration

Finally! That felt like it was never going to end.


# 7.3.3.1 A kernel module is not loaded automatically

This is interesting, but a bit confusing how to actually see these things for
yourself. I don't see any `modalias` files in `/sys/bus` on my host system or
in the chroot system. And I don't know what to pass to `modinfo` to try out
what these sections are describing.

# 7.4.1.1 Disabling Persistent Naming on the Kernel Command Line

`net.ifnames=0` passed on the kernel command line uses the old style `eth0`,
`eth1`, `etc` names though those can have timing issues.

# 7.4.1.2 Creating Custom Udev Rules

Ah now this is useful, this would be nice to get cleaner names than the auto
generated ones.

The script doesn't generate anything though...

This is probably different in systemd land as well, so not sure how useful this
is.

# 7.5.1 Creating Network Interface Configuration Files

Why is it going through all this configuration information before booting the
system? Skipping this since I don't know the names of the interfaces.


# 7.5.2 Creating the /etc/resolve.conf File

```
nameserver 192.168.1.1
nameserver 1.1.1.1
```

# 7.5.3 Configuring the system hostname

```shell
echo "lfs" > /etc/hostname
```

# 7.5.4 Customizing the /etc/hosts File

```
127.0.0.1 localhost
127.0.1.1 lfs.localdomain lfs
::1 localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
```

# 7.6.4 Configuring the System Clock

Their `/etc/rc.d/init.d/setclock` script reads from `/etc/sysconfig/clock`.

This script is triggered by `udev` via the configuration in
`/etc/udev/rules.d/55-lfs.rules`.

```
# This causes the system clock to be set as soon as /dev/rtc becomes available.
SUBSYSTEM=="rtc", ACTION=="add", MODE="0644", RUN+="/etc/rc.d/init.d/setclock start"
KERNEL=="rtc", ACTION=="add", MODE="0644", RUN+="/etc/rc.d/init.d/setclock start"
```

# 7.6.5 Configuring the Linux Console

Kind of confusing, it seems like `/etc/sysconfig/console` is required, but I
think it isn't actually required if you are okay with the defaults, ASCII only
and US keyboard. So I'm not setting anything in this file.

# 7.6.8 The rc.site File

Okay so `rc.site` can override defaults, but where do the defaults come from?

Looking at `#DISTRO="Linux From Scratch"` I found where it is being actually
defaulted in `/etc/init.d/rc`.

```console
(lfs chroot) root:/etc/init.d# grep DISTRO= -r *
rc:DISTRO=${DISTRO:-"Linux From Scratch"}
(lfs chroot) root:/etc/init.d# file rc
rc: Bourne-Again shell script, ASCII text executable
```

Which is the whole entry point for the LFS init system as set in
`/etc/inittab`, so I think this is making some sense to me now.

# 7.6.8.1 Customizing the Boot and Shutdown Scripts

Show me the filesystem check logs

`VERBOSE_FSCK=yes`

# 7.7 The Bash Shell Startup Files

```console
(lfs chroot) root:/etc/init.d# cat /etc/profile
export LANG=en_US.iso88591
```

# 7.8 Creating the /etc/inputrc File

vi mode supremecy + some modern niceties

```
set input-meta on
set convert-meta off
set output-meta on
set bell-style none

set keyseq-timeout 0
set editing-mode vi
set keymap vi-command
set completion-ignore-case on
set show-mode-in-prompt on
set vi-ins-mode-string \1\e[0;32m\2ins \1\e[0m\2
set vi-cmd-mode-string \1\e[43;30m\2cmd \1\e[0m\2
set emacs-mode-string \1\e[0;31m\2@ \1\e[0m\2
set show-all-if-ambiguous on
set colored-stats on
set colored-completion-prefix on

$if mode=vi
set keymap vi-command
# these are for vi-command mode
Control-l: clear-screen

set keymap vi-insert
# these are for vi-insert mode
Control-l: clear-screen

# map C-p and C-n for insert mode
Control-p: previous-history
Control-n: next-history
$endif
```

# Chapter 8 Making the LFS System Bootable

Alllmoosst there!

# 8.1 Introduction

I already have a grub config on this drive at `/dev/sda1`, so I think I'll just
be needing to tweak that instead of following the LFS GRUB instructions
exactly.

# 8.2 Creating the /etc/fstab file

```
# file system  mount-point  type     options             dump  fsck
/dev/sda3      /            ext4     defaults            1     1
/dev/sda7      swap         swap     pri=1               0     0
proc           /proc        proc     nosuid,noexec,nodev 0     0
sysfs          /sys         sysfs    nosuid,noexec,nodev 0     0
devpts         /dev/pts     devpts   gid=5,mode=620      0     0
tmpfs          /run         tmpfs    defaults            0     0
devtmpfs       /dev         devtmpfs mode=0755,nosuid    0     0
```

# 8.3.1 Installation of the kernel

Okay we'll skip `make defconfig` since the host machine is different than the
target machine and go straight for `make menuconfig`. I remember doing this for
a gentoo install like 8 years ago. Just going to skim through for options that
make sense to enable or disable.

General Setup:
  Default Hostname: lfs

The documentation's `Kernel hacking` option does not seem to be specified
correctly.

Kernel hacking -> *x86 Debugging* ->
  Choose kernel unwinder: Frame pointer unwinder

I am not using UEFI on the target machine, so let's try removing support for
it.

Processor type and features ->
  EFI stub support: uncheck

Processor type and features ->
  Support for extended (non-PC) x86 platforms: uncheck

Networking support ->
  Amateur Radio support: uncheck

Device Drivers -> x86 Platform Specific Device Drivers:
  Eee PC Hotkey Driver: uncheck

Wondering if I can boot and then do a `make defconfig` to make it figure out
things for me?

Security options ->
  NSA SELinux Support: uncheck

Processor type and features -> Supported processor vendors ->
  Only check Intel, since that is what this is going on. Okay nevermind I can't
  change these?

I think there are generic options set for things I'll be needing?

Saved config as `first-try.config`.

Let's build this! `make -j8`. Only took a few minutes!

`cp -iv arch/x86_64/boot/bzImage /boot/vmlinuz-5.5.3-lfs-9.1`

Going to keep the kernel sources around, might need to recompile a new kernel.

`chown -R 0:0 /sources/linux-5.5.3`

# 8.4.1 Introduction

Okay mounting the `/boot` partition (`/dev/sda1`) from the external drive to
the host machine in order to modify the grub config to add a new entry.

```
sudo mount /dev/sda1 /storage/mnt/x200-boot
```

# 8.4.4 Creating the GRUB Configuration File

Added an entry to my existing boot config
(`/storage/mnt/x200-boot/grub/libreboot_grub.cfg`)

```
menuentry 'lfs  [2]' --hotkey='2' {
        set root='(ahci0,3)'
        linux /boot/vmlinuz-5.5.3-lfs-9.1 root=/dev/sda3
}
```

I am not sure why they have `ro` set in the documentation. Sifting around for
the docs on the kernel parameters that does mean that root will be read-only on
boot, which I would think would make the system basically unusable? I don't see
any references later in the documentation to remove that.

```console
(lfs chroot) root:/sources/linux-5.5.3# find . -name '*kernel*param*'
./Documentation/ABI/testing/sysfs-kernel-boot_params
./Documentation/admin-guide/kernel-parameters.rst
./Documentation/admin-guide/kernel-parameters.txt
./Documentation/translations/it_IT/admin-guide/kernel-parameters.rst
(lfs chroot) root:/sources/linux-5.5.3# egrep '^\sro\s.*KNL' Documentation/admin-guide/kernel-parameters.txt
        ro              [KNL] Mount root device read-only on boot
```

I'm gonna *not* set that, since it doesn't make any sense.

# 9.3 Rebooting the System

Not going to install anything extra! Hoping it will boot and I'll set up static
networking, then get whatever else I need.

...Ah this thing won't even have `curl` or `wget` Or ssh to remote into it or
transfer files to it. I suppose I can download things onto a flash drive and
transfer it that way (or boot into the main OS on the box, mount the lfs drive
and write whatever files to it). Alternatively there are some interesting
approaches here
https://stackoverflow.com/questions/1121003/a-command-to-download-a-file-other-than-wget

(Edit from later: I realize now `inetutils` provides `ftp`, so that is
definitely usable for getting more software out of the box without any new
packages and transfering things around on a local network.)

I am anxious to try and boot this, so I am continuing on.

Umounting my other boot partition in addition to the LFS docs.

```
sudo umount /storage/mnt/x200-boot
```

Holy cow I put the drive back in and it actually booted! That is a relief.

I was the 28,479th person to register on
http://www.linuxfromscratch.org/cgi-bin/lfscounter.php as being an LFS user :).

It looks like the boot process takes about 3 seconds, very nice.

# Beyond Linux from Scratch

Now to set up networking, then on to Beyond Linux from Scratch to get some more
software
http://www.linuxfromscratch.org/blfs/downloads/9.1/BLFS-BOOK-9.1-nochunks.html

Off the top of my head here are some things I'm interested in that are in BLFS:
 - OpenSSH
 - wget
 - curl
 - lynx
 - rust toolchain

Here are some things that are not in BLFS to start making this into a basically
daily driver level for me:
 - tmux
 - sway if I want to start getting into the wild world of non-tty's
 - st or alacritty terminal emulator
 - badwolf web browser https://hacktivis.me/projects/badwolf
 - syncthing
 - keepassxc

