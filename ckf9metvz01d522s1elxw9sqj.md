## Hardening OpenWRT - adding non-root user account

I have at home an OpenWRT on TP-Link 1043ND. As I'm most of the time out of my home, I use this for two functions:
 - as reliable WiFi Access Point
 - as last resort remote access device
 - as a box used to wake up other Ethernet enabled devices @ home.

As this box is exposed to Internet via IPv6 address, I decided to harden it a little.

1. Adding extra non privileged user account:

```
root@OpenWrt:~# opkg update

root@OpenWrt:~# opkg install shadow-useradd
root@OpenWrt:~# mkdir /home
root@OpenWrt:~# useradd -D
GROUP=100
HOME=/home
INACTIVE=-1
EXPIRE=
SHELL=
SKEL=/etc/skel
CREATE_MAIL_SPOOL=no

root@OpenWrt:~# useradd -m -s /bin/ash bart

root@OpenWrt:~# cat /etc/passwd
root:x:0:0:root:/root:/bin/ash
daemon:*:1:1:daemon:/var:/bin/false
ftp:*:55:55:ftp:/home/ftp:/bin/false
network:*:101:101:network:/var:/bin/false
nobody:*:65534:65534:nobody:/var:/bin/false
bart:x:1000:1000::/home/bart:/bin/ash

root@OpenWrt:~# passwd bart
Changing password for bart
New password:
Retype password:

Password for bart changed by root
```

Now I have an user account that I can login to remotely and it is not root.