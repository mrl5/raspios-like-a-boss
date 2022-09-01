# Setting up Raspberry Pi OS in terminal like a boss

![like a boss](https://as2.ftcdn.net/v2/jpg/02/25/53/75/500_F_225537518_9UhVyvJ9Za8uKIyosc3Mboonj6nGVE5V.jpg)


## Download

* UI link: https://www.raspberrypi.com/software/operating-systems/
* old school link: https://downloads.raspberrypi.org/raspios_lite_arm64/images/ **<=== use this one - like a boss**

```console
wget https://downloads.raspberrypi.org/raspios_lite_arm64/images/raspios_lite_arm64-2022-04-07/2022-04-04-raspios-bullseye-arm64-lite.img.xz
wget https://downloads.raspberrypi.org/raspios_lite_arm64/images/raspios_lite_arm64-2022-04-07/2022-04-04-raspios-bullseye-arm64-lite.img.xz.sha256
```


## Ensure checksum and unpack

```console
IMGXZ=$(ls *raspios-*.img.xz)
CHECKSUM=${IMGXZ}.sha256

sha256sum -c $CHECKSUM && unxz $IMGXZ
```


## Transfer to MMC

Using `dd` aka (d)isc (d)estroyer - better check twice the value that you put
to `of=`

```console
IMG= # e.g. 2022-04-04-raspios-bullseye-arm64-lite.img
MMC= # e.g. /dev/sdd

sudo dd if="$IMG" of="$MMC" bs=4M conv=fsync status=progress
```

This will freeze although there is `status=progress`. The reason and solution
to see the progress is described in https://superuser.com/a/1406378

So open new terminal and
```
watch grep -i dirty /proc/meminfo
```
The value should decrease


## (optional) Mount filesystem

Lets transfer ssh pubkey so that it can be added later to `.ssh/authorized_keys`

```console
mkdir -p /mnt/sdcard
mount $MMC /mnt/sdcard
cp -v /home/your_user/.ssh/id_rsa.pub /mnt/sdcard/root/
```

## Plug in Raspberry Pi

I don't know how to bypass this step so that I can ssh right away so ...

1. Put MMC to you Raspberry Pi
2. Connect HDMI
3. Connect keyboard
4. Power up Raspberry Pi
5. Wait for dialog
6. Create your user

If at some point you see black page you can always `CTRL+ALT+F1` but then make
sure to find the dialog by switching to different TTYs - each `Fx` key is
different TTY (so `F1` is TTY1, `F2` is TTY2 etc.)


## SSH + basic security

```console
sudo raspi-config
```
Checklist:
1. Disable autologin
2. Disable ssh (for now)
3. Configure timezone

Create dedicated ssh group:
```console
sudo groupadd ssh-users
sudo usermod -a -G ssh-users your_user
```

Add pubkey to `.ssh/authorized_keys`
```console
mkdir -m 700 ~/.ssh
touch ~/.ssh/authorized_keys && chmod 600 $_
sudo cat /root/id_rsa.pub > ~/.ssh/authorized_keys
```

Modify `/etc/ssh/sshd_config`
```diff
--- /etc/ssh/sshd_config	2021-03-13 10:59:40.000000000 +0100
+++ /etc/ssh/sshd_config	2022-09-01 02:39:27.521208214 +0200
@@ -31,12 +31,13 @@
 # Authentication:
 
 #LoginGraceTime 2m
-#PermitRootLogin prohibit-password
+PermitRootLogin no
 #StrictModes yes
 #MaxAuthTries 6
 #MaxSessions 10
 
-#PubkeyAuthentication yes
+PubkeyAuthentication yes
+AllowGroups ssh-users
 
 # Expect .ssh/authorized_keys2 to be disregarded by default in future.
 #AuthorizedKeysFile	.ssh/authorized_keys .ssh/authorized_keys2
@@ -55,7 +56,7 @@
 #IgnoreRhosts yes
 
 # To disable tunneled clear text passwords, change to no here!
-#PasswordAuthentication yes
+PasswordAuthentication no
 #PermitEmptyPasswords no
 
 # Change to yes to enable challenge-response passwords (beware issues with
@@ -83,16 +84,16 @@
 # If you just want the PAM account and session checks to run without
 # PAM authentication, then enable this but set PasswordAuthentication
 # and ChallengeResponseAuthentication to 'no'.
-UsePAM yes
+UsePAM no
 
 #AllowAgentForwarding yes
 #AllowTcpForwarding yes
 #GatewayPorts no
-X11Forwarding yes
+X11Forwarding no
 #X11DisplayOffset 10
 #X11UseLocalhost yes
 #PermitTTY yes
-PrintMotd no
+PrintMotd yes
 #PrintLastLog yes
 #TCPKeepAlive yes
 #PermitUserEnvironment no
```

Require password for `sudo`
```console
sudo visudo /etc/sudoers.d/010_pi-nopasswd
```
```diff
--- /etc/sudoers.d/010_pi-nopasswd  2022-09-01 01:50:50.160023746 +0100
+++ /etc/sudoers.d/010_pi-nopasswd  2022-09-01 01:50:50.160023746 +0100
@@ -1 +1 @@
-#your_user ALL=(ALL) NOPASSWD: ALL
+your_user ALL=(ALL) NOPASSWD: ALL
```

Disable logins by password
```console
sudo passwd -ld root
```

Disable bluetooth
```console
sudo systemctl disable bluetooth
```

Remove not needed features from `/boot/config.txt`:
```diff
--- config.txt	2022-09-01 16:16:02.000000000 +0200
+++ config.txt	2022-09-01 16:18:10.000000000 +0200
@@ -50,10 +50,10 @@
 # Additional overlays and parameters are documented /boot/overlays/README
 
 # Enable audio (loads snd_bcm2835)
-dtparam=audio=on
+dtparam=audio=off
 
 # Automatically load overlays for detected cameras
-camera_auto_detect=1
+camera_auto_detect=0
 
 # Automatically load overlays for detected DSI displays
 display_auto_detect=1
@@ -81,3 +81,4 @@
 arm_boost=1
 
 [all]
+dtoverlay=pi3-disable-bt
```

Configure WiFi and enable ssh
```console
sudo raspi-config
```

Upgrade the system
```console
sudo apt-get update && sudo apt-get upgrade
```

Reboot
```console
sudo reboot
```


## Additional configuration

SSH into your rpi and lets setup unattended upgrades
```console
sudo apt-get install unattended-upgrades apt-listchanges apticron
```
```diff
--- /etc/apt/apt.conf.d/50unattended-upgrades	2021-02-19 13:11:42.000000000 +0100
+++ /etc/apt/apt.conf.d/50unattended-upgrades.new	2022-09-01 16:51:30.027878341 +0200
@@ -91,7 +91,7 @@
 // If empty or unset then no email is sent, make sure that you
 // have a working mail setup on your system. A package that provides
 // 'mailx' must be installed. E.g. "user@example.com"
-//Unattended-Upgrade::Mail "";
+Unattended-Upgrade::Mail "root";
 
 // Set this value to one of:
 //    "always", "only-on-error" or "on-change"
```
```diff
--- /dev/null	2022-09-01 16:44:19.491999997 +0200
+++ 02periodic	2022-09-01 16:54:59.446369891 +0200
@@ -0,0 +1,6 @@
+APT::Periodic::Enable "1";
+APT::Periodic::Update-Package-Lists "1";
+APT::Periodic::Download-Upgradeable-Packages "1";
+APT::Periodic::Unattended-Upgrade "1";
+APT::Periodic::AutocleanInterval "1";
+APT::Periodic::Verbose "2";
```
```console
sudo unattended-upgrades -d
```
