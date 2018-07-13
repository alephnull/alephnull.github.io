---
layout: post
title:  "Setting up time machine on Debian"
#date:   2018-06-25 10:50:58 +07
categories: [x-compile, arm, debian, time-machine]
---

Time machine backups used to happen on the AC3200 as it has a recent [netatalk](http://netatalk.sourceforge.net/). This works but the AC3200 does not have much spare memory and the disk that hosted the backups was also used by aria2c. aria2c on the router used a fair bit of RAM and CPU which I could ill afford. Hence both aria2c and time machine had to be moved to the cubox.

The netatalk in Debian is [ancient](http://bugs.debian.org/cgi-bin/bugreport.cgi?bug=690227) and doesn't seem to be getting better. Adrian Knoth has an [updated netatalk](https://github.com/adiknoth/netatalk-debian) package that we can use. I could compile on the cubox itself but this seemed like a job for a cross compiler.

Start by

```
$ sudo dpkg --add-architecture armhf
$ sudo apt-get update
$ inst crossbuild-essential-armhf
```

Build the package with

```
$ sudo apt-get -a armhf build-dep netatalk
$ inst libevent-dev:armhf
$ dpkg-buildpackage -uc -us -b --host-arch armhf
```

Here I found that cross compilation can be hairy and the Debian infra is rather dated. I wanted to learn the newer `dh_*` tools and so I started from scratch with the [NM guide](https://www.debian.org/doc/manuals/maint-guide/dreq.en.html)

## Links

* [Building netatalk v3 for Debian 9 (Stretch)](http://netatalk.sourceforge.net/wiki/index.php/Install_Netatalk_3.1.11_on_Debian_9_Stretch)
* Daniel Lange's [guide](https://daniel-lange.com/archives/102-Apple-Timemachine-backups-on-Debian-8-Jessie.html) for Debian 8 (Jessie)
