---
layout: post
title:  "Building a Digital Homestead: Chapter 1"
date:   Sat, 10 Nov 2018 20:04:10 +07
categories: [homestead, buildroot, vagrant]
---

_At first I hoped that such a technically unsound project would collapse but I soon realized it was doomed to success. Almost anything in software can be implemented, sold, and even used given enough determination. There is nothing a mere scientist can say that will stand against the flood of a hundred million dollars. But there is one quality that cannot be purchased in this way — and that is reliability. The price of reliability is the pursuit of the utmost simplicity. It is a price which the very rich find most hard to pay._
 -- C. A. R. Hoare, on PL/1

# Problem statement

The HC2s can only boot from the SD card. The SD card is painfully slow. Having to update individual OS installs all the time would add overhead. A proposed solution is to PXE boot the hardware and use an NFS root system. This way updates only have to be done once and can be available to all.

To complete the boot, a root filesystem image will have to be provided to the hardware. There are a few options for building this rootfs:
- [EmDebian](http://www.emdebian.org/)
- [buildroot](https://buildroot.org/)
- [yocto](https://www.yoctoproject.org/)

# EmDebian

While it would be good to have Debian on the HC2s, this looks like a fair bit of work.

# Yocto

While this appears rather widely used, nothing that I read for a day made sense to me.

# Buildroot

This was spoken of as having the lowest barrier of entry, so off we go.

Buildroot provides a [Vagrant file](https://buildroot.org/downloads/Vagrantfile) to quickly get started. Installed vagrant and vagrant-lxc from Debian and attempting to use this file and we run into our first problem:

```
Bundler, the underlying system Vagrant uses to install plugins,
reported an error. The error is shown below. These errors are usually
caused by misconfigured plugin installations or transient network
issues. The error from Bundler is:
 
conflicting dependencies ttfunk (= 1.5.1) and ttfunk (~> 1.4.0)
  Activated ttfunk-1.4.0
  which does not match conflicting dependency (= 1.5.1)
 
  Conflicting dependency chains:
    prawn (= 2.1.0), 2.1.0 activated, depends on
    ttfunk (~> 1.4.0), 1.4.0 activated
 
  versus:
    ttfunk (= 1.5.1)
 
  Gems matching ttfunk (= 1.5.1):
    ttfunk-1.5.1
```

After configuring `lxc` to use my existing `libvirt-bin` bridge and some blind hacking on the `Vagrantfile`, I got to the stage that it would attempt to pull the image `bento/ubuntu-16.04` cannot be used with lxc. Now I try to install the `vagrant-mutate` plugin and I come up against the same error as above. Looking at the BTS leads me to [#896904](https://bugs.debian.org/cgi-bin/bugreport.cgi?bug=896904) which has a similar error message. Following the Github issues [dotless-de/vagrant-vbguest#292](https://github.com/dotless-de/vagrant-vbguest/issues/292) and [devopsgroup-io/vagrant-hostmanager#256](https://github.com/devopsgroup-io/vagrant-hostmanager/issues/256) leads me to the suggestion that I install vagrant from [Hashicorp's repository](https://releases.hashicorp.com/vagrant/). 

At this point, I give up on vagrant. I figure I can do all this in an lxc.

## LXC

Turns out that Debian only has LXC 2.0.9 and [does not have lxd](https://bugs.debian.org/cgi-bin/bugreport.cgi?bug=768073) at all. Never mind, I just need a chroot.

Recently, I had to enable unprivileged user namespaces for Chromium. The same reasoning applies to lxc, escaping the container results in root on the host. My user is sudo'd up to the hilt that there is no real difference between my user and root. But it is principle of the thing. To configure sub{u,g}ids, pick a random number and add 65536 to it. I picked 100000 and thus ended up with 165536.

```
# apt install uidmap
# usermod --add-subuids 100000-165536 $USER
# usermod --add-subgids 100000-165536 $USER
# # cat > 80-lxc-userns.conf
kernel.unprivileged_userns_clone=1
^D
# sysctl --system
$ echo "$USER veth lxcbr0 10"| sudo tee -i /etc/lxc/lxc-usernet
```

Now add the following to `~/.config/lxc/default.conf`. Make sure the sub{u,g}ids match the numbers above.

```
lxc.include = /etc/lxc/default.conf
# Subuids and subgids mapping
lxc.id_map = u 0 100000 65536
lxc.id_map = g 0 100000 65536
# "Secure" mounting
lxc.mount.auto = proc:mixed sys:ro cgroup:mixed
```

Create the container as below. Note that you cannot use the on-disk ubuntu template with unprivileged LXC. At this point, I didn't really want to know.

```
$ lxc-create -n buildroot --dir=$PWD/buildroot -t download -- --dist ubuntu -r xenial -a amd64
```

If you get the error that lxc cannot write to `~/.cache`, add an ACL for the subuid that was used above.

```
$ setfacl -m u:100000:rwx .cache/
```

The next problem is that the container cannot start in unprivileged mode. Get verbose logs with 

```
$ lxc-start -n buildroot -l trace -o u.log
```

The first error is `lxc_utils - utils.c:mkdir_p:257 - Permission denied - failed to create directory '/sys/fs/cgroup/freezer/lxc'`. This leads me to [some patches that are in Ubuntu](https://discuss.linuxcontainers.org/t/failed-creating-cgroups/272/9) or [various](https://github.com/lxc/lxc/issues/1205) [workarounds](https://www.linuxquestions.org/questions/linux-kernel-70/lxc-unprivileged-container-in-debian-jessie-cgroups-permissions-4175540174/). 

Perhaps, just a good ol' schroot will at least let me get started on buildroot. Long term, it appears that Ubuntu might be a better choice for development.

## schroot

```
$ mkdir xenial
$ fakeroot debootstrap xenial ./xenial http://archive.ubuntu.com/ubuntu
```

Let it chug along for a while, then realise that debootstrap requires sudo.

```
$ sudo debootstrap xenial ./xenial http://archive.ubuntu.com/ubuntu
```

Create a file `/etc/schroot/chroot.d/xenial.conf` with:

```
[xenial]
description=Xenial Buildroot
directory=/home/alok/var/chroot/xenial
root-users=alok
users=alok
type=directory
```

Create a schroot session with,

```
$ schroot -b -c xenial -n buildroot
```

Enter that session with,

```
$ schroot -r -c xenial
```

You can become root with

```
$ schroot -r -c xenial -u root
```

Or setup sudo inside the chroot.

Finally, we can get started with buildroot.

## Buildroot setup

Use Ubuntu's mirror protocol and add universe to `/etc/apt/sources.list`. Something like,

```
deb mirror://mirrors.ubuntu.com/mirrors.txt xenial main universe
```

Then,

```
# dpkg --add-architecture i386
# apt update
# apt purge -q snapd lxcfs lxd ubuntu-core-launcher snap-confine
# apt install build-essential libncurses5-dev git bzr cvs mercurial subversion libc6:i386 unzip bc wget
```

117 kBps. Gosh, my internet sucks.

Get the latest buildroot image.

```
$ wget -q -c http://buildroot.org/downloads/buildroot-2018.08.tar.gz
$ tar xaf buildroot-2018.08.tar.gz
```

`make nconfig` followed by `make` will build an image.

So endeth the lesson.
