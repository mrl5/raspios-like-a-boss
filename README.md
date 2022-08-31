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
