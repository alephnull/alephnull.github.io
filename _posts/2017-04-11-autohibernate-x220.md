---
layout: post
title:  "Hibernate on low-battery on the Thinkpad x220"
date:   2017-04-11 10:31:52 +0700
categories: [x220, hibernate, debian]
---

When I first got this laptop in 2012, Debian would hibernate the laptop when the battery went too low. I didn't set this up and it worked well until it didn't work. 

On the X220, the battery LED is not visible to you when you are using the laptop, it is on the top side of the lid. 

Figuring this out would mean a lot of waiting and watching the output of `acpi_listen` and `udevadm monitor`. The alternative would be to write a cronjob that would check battery status. This workaround offended my sensibilities and I therefore endured unexpected shutdowns. This became steadily worse as the battery capacity degraded. 

But not today. I found user [svenper](https://bbs.archlinux.org/viewtopic.php?id=216225) on the arch forums who figured out the fact that the firmware does send an ACPI event. This event is sent when,

- ac adapter is plugged in
- the battery is at 80%
- the battery is at 20%
- the battery is at 5%

To adapt his solution to Debian's `acpid`, create a file `/etc/acpi/events/thinkpad-low-battery` with contents:

```
event=battery PNP0C0A:00 00000080 00000001
action=/etc/acpi/tp-low-battery.sh
```

And `/etc/acpi/tp-low-battery.sh` is:

```
#!/bin/sh

BATTERY=BAT0

CAPACITY=$(cat /sys/class/power_supply/${BATTERY}/capacity)
STATUS=$(cat /sys/class/power_supply/${BATTERY}/status)
/usr/bin/logger -t auto-hibernate -p info Got event $STATUS ${CAPACITY}%
if [ $CAPACITY -le 6 ] && [ $STATUS = Discharging ]
then
    /usr/bin/logger -t auto-hibernate Hibernating due to low battery (${CAPACITY})
    /usr/bin/systemctl hibernate
fi
```

Test this with by restarting acpid and plugging in the ac adapter. You should see some output in `/var/log/messages`.

Thanks svenper!
