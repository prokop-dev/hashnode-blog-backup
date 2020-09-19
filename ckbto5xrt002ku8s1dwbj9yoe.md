## Complete upgrade of EdgeRouter ER-X

After logged to my ER-X, I saw the following greeting:

```
Boot image can be upgraded to version [ e50_002_4c817 ].
Run "add system boot-image" to upgrade boot image.
``` 

What actually triggered the action: Let's ensure that the firmware is really up-to-date. 

# Upgrade the boot-loader
The EdgeRouter bootloader controls functions such as the LED boot behavior, configuration/driver loading and much more. First some investigation:

```
bart@ubnt:~$ show system boot-image
The system currently has the following boot image installed:
Current boot version: UNKNOWN
Current boot md5sum : dbce6273c8740383f84040e33f7fffe7

New uboot version is available: boot_e50_002_4c817.tar.gz
New boot md5sum : 152b37ac18d23006c1787ed8920c1ea2
Run "add system boot-image" to upgrade boot image.
```

Then run the following command in order to update the bootloader:

```
bart@ubnt:~$ add system boot-image
Uboot version [UNKNOWN] is about to be replaced
Warning: Don't turn off the power or reboot during the upgrade!
Are you sure you want to replace old version? (Yes/No) [Yes]: yes
Preparing to upgrade...Done
Copying upgrade boot image...Done
Checking boot version: Current is UNKNOWN; new is e50_002_4c817 ...Done
Checking upgrade image...Done
Writing image...Boot image has been upgraded.
Reboot is needed in order to apply changes!
Done
Upgrade boot completed
```

Then simply reboot your router. There should be no more nagging "MotD" when you login to your router. You can also check if your changes were effective:

```
bart@ubnt:~$ show system boot-image
The system currently has the following boot image installed:
Current boot version: e50_002_4c817
Current boot md5sum : 152b37ac18d23006c1787ed8920c1ea2
```

# How to upgrade router firmware

Next we'll perform remote firmware upgrade over SSH.

First let's check current firmware version:

```
bart@ubnt:~$ show version
Version:      v2.0.8
Build ID:     5247496
Build on:     11/20/19 11:24
Copyright:    2012-2019 Ubiquiti Networks, Inc.
HW model:     EdgeRouter X 5-Port
HW S/N:       788A2009772D
Uptime:       17:36:43 up 6 min,  1 user,  load average: 1.03, 1.00, 0.53
```

This should be contrasted with latest firmware available on UniFi download page. Let's also note the checksum of the firmware we're going upgrade to. As we are doing the remote upgrade without physical access to device (i.e. I'm 5000 miles away from my router) we're going to do this in safest possible way:

1. We will download firmware first to the router temporary filesystem (/tmp being in fact a ramdisk).
2. We will validate checksum.
3. We will flash firmware locally.

The latest firmware when performing the procedure is:
![xx01.PNG](https://cdn.hashnode.com/res/hashnode/image/upload/v1593021935382/PhgS5XlWt.png)

Now, we check if there is enough filesystem space to download firmware. We're looking into /tmp spare space:

```
bart@ubnt:~$ show system storage
Filesystem                Size      Used Available Use% Mounted on
ubi0_0                  214.9M    140.4M     69.8M  67% /root.dev
overlay                 214.9M    140.4M     69.8M  67% /
devtmpfs                123.0M         0    123.0M   0% /dev
tmpfs                   123.6M         0    123.6M   0% /dev/shm
tmpfs                   123.6M      2.3M    121.3M   2% /run
tmpfs                     5.0M         0      5.0M   0% /run/lock
tmpfs                   123.6M         0    123.6M   0% /sys/fs/cgroup
tmpfs                   123.6M         0    123.6M   0% /run/shm
tmpfs                   123.6M      8.0K    123.6M   0% /tmp
tmpfs                   123.6M         0    123.6M   0% /lib/init/rw
tmpfs                   123.6M     84.0K    123.6M   0% /var/log
none                    123.6M    476.0K    123.2M   0% /opt/vyatta/config
tmpfs                    24.7M         0     24.7M   0% /run/user/1001
```

124 MB of space available is enough to download the firmware to ramdisk. Standard firmware has no ```wget```, but there is ```curl``` available.

```
bart@ubnt:~$ curl https://dl.ui.com/firmwares/edgemax/v2.0.8-hotfix.1/ER-e50.v2.0.8-hotfix.1.5278088.tar > /tmp/ER-e50.v2.0.8-hotfix.1.5278088.tar
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 80.6M  100 80.6M    0     0  3940k      0  0:00:20  0:00:20 --:--:-- 3888k
```

It is very important to compare the checksum for downloaded router firmware:

```
bart@ubnt:~$ sha256sum /tmp/ER-e50.v2.0.8-hotfix.1.5278088.tar
b877aa2404ec768c2000d15c2aea53205be5f64046f671ba37d67860cd582846  /tmp/ER-e50.v2.0.8-hotfix.1.5278088.tar
```

We are all set to begin firmware upgrade now. First, let's inspect what's in flash currently:

```
bart@ubnt:~$ show system image
The system currently has the following image(s) installed:

v2.0.8.5247496.191120.1124     (running image) (default boot)
v1.7.1.4821926.151103.1114
```

Apparently there is old firmware 1.7.1 sitting there. It is recommended to remove it to make a space in flash for a new firmware (note ER-X has quite limited flash so you cannot have more than two images in flash.

```
bart@ubnt:~$ delete system image
The system currently has the following image(s) installed:

v2.0.8.5247496.191120.1124     (running image) (default boot)
v1.7.1.4821926.151103.1114

You are about to delete image [v1.7.1.4821926.151103.1114]
Are you sure you want to delete ? (Yes/No) [Yes]: yes
Removing old image... Done
```

Let's now check if we really have more space available - please compare below result with ```show system storage``` result above.

```
bart@ubnt:~$ df -h
Filesystem                Size      Used Available Use% Mounted on
ubi0_0                  214.9M     77.3M    132.9M  37% /root.dev
overlay                 214.9M     77.3M    132.9M  37% /
```

We have now 133 MB of available flash space. Let's flash the new firmware:

```
bart@ubnt:~$ add system image /tmp/ER-e50.v2.0.8-hotfix.1.5278088.tar
Checking upgrade image...Done
Preparing to upgrade...Done
Clearing directory /var/cache/apt (1.1M)...Done
Copying upgrade image...Done
Removing old image...Done
Checking upgrade image...Done
Copying config data...Done
Finishing upgrade...Done
Upgrade completed
```

How let's check if new firmware is selected to boot into after the restart.

```
bart@ubnt:~$ show system image
The system currently has the following image(s) installed:

v2.0.8-hotfix.1.5278088.200305.1641 (default boot)
v2.0.8.5247496.191120.1124     (running image)
```

A reboot is needed to boot info newly flashed image.

# Final notes

That is all, EdgeRouter ER-X is now running the latest boot-image and firmware

I recommend to leave the previous firmware and not to issue ```delete system image``` after the upgrade. It allows to "block" the flash space for future upgrades. We should remember that space is limited on EX-R and installing too many packages would prevent future upgrades.