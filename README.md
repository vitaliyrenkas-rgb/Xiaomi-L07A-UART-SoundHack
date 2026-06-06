# Xiaomi AI Speaker L07A — UART root + custom sound pack

## Що це

Це нотатки по Xiaomi / Redmi AI Speaker L07A, де вдалося:

* зайти в UART через штатний 3-pin JST-XH 2.54 debug connector;
* зупинити U-Boot;
* тимчасово завантажити Linux з `init=/bin/sh`;
* отримати root shell без підбору DSA-пароля;
* знайти writable UDISK partition;
* витягнути оригінальний sound pack;
* замінити китайські голосові prompt-и на кастомні системні мелодії;
* зробити це без перепрошивки rootfs і без порушення secure boot.

Метод не патчить squashfs rootfs. Заміна звуків робиться через штатний writable symlink `/tmp/udisk/sound`.

## Hardware

Tested device:

* Model: Xiaomi / Redmi AI Speaker L07A
* SoC: Allwinner R328 / sun8iw18
* RAM: 64 MiB
* NAND: SPI NAND
* Rootfs: squashfs read-only
* Writable partition: `/dev/nand0p9`, mounted as EXT4

UART connector:

* Connector: JST-XH 2.54, 3-pin
* UART level: 3.3V
* Baudrate: 115200 8N1

Pinout used:

1. W / TX — board TX
2. B / GND — ground
3. G / RX — board RX

USB-TTL connection:

Board TX -> USB-TTL RX
Board RX -> USB-TTL TX
Board GND -> USB-TTL GND

Do not connect USB-TTL VCC / 3V3 / 5V to the board.

Power the speaker from its normal 12V input.

## UART / U-Boot access

Open UART terminal at:

115200 8N1, no flow control

Power-cycle the speaker and interrupt autoboot when U-Boot shows:

Hit any key to stop autoboot:  0

U-Boot prompt:

=>

Useful safe commands:

help

version

printenv

Confirmed U-Boot version:

U-Boot 2018.05 (Dec 11 2019 - 02:53:20 +0000) Allwinner Technology, Build: jenkins-Mico_l07a_ota_publish-63

Important environment values:

boot_first=sunxi_flash read 40007800 ${boot_partition_1};bootm 40007800

boot_partition_1=kernel1

boot_partition_2=kernel2

first_root=/dev/nand0p3

second_root=/dev/nand0p5

partitions=env@nand0p1:kernel1@nand0p2:rootfs1@nand0p3:kernel2@nand0p4:rootfs2@nand0p5:misc@nand0p6:private@nand0p7:crashlog@nand0p8:UDISK@nand0p9

Active boot path:

kernel1 + rootfs1

rootfs1 = /dev/nand0p3

## Temporary root shell via init=/bin/sh

Do not run `saveenv`. The following change is temporary and only affects this boot.

In U-Boot:

run setargs_first

Check current bootargs:

printenv bootargs

Set temporary shell init:

setenv bootargs earlyprintk=${earlyprintk} console=${console} root=${first_root} rootwait init=/bin/sh rdinit=${rdinit} loglevel=8 partitions=${partitions} gpt=${gpt} rotpk_status=${rotpk_status}

Verify:

printenv bootargs

Expected part:

init=/bin/sh

Boot:

run boot_first

Expected result after Linux boots:

BusyBox v1.27.2 () built-in shell (ash)

/bin/sh: can't access tty; job control turned off

Root shell prompt:

/ #

Confirm root:

id

Expected:

uid=0(root) gid=0(root)

## Mount proc/sys

Because `/sbin/init` was bypassed, `/proc` and `/sys` may not be mounted.

Run:

mount -t proc proc /proc

mount -t sysfs sysfs /sys

Optional devtmpfs mount may fail with `Resource busy`; that is OK:

mount -t devtmpfs devtmpfs /dev

Check command line:

cat /proc/cmdline

Expected to include:

root=/dev/nand0p3 init=/bin/sh

## Partition map

Check NAND block devices:

ls -la /dev/nand*

Expected:

/dev/nand0
/dev/nand0p1
/dev/nand0p2
/dev/nand0p3
/dev/nand0p4
/dev/nand0p5
/dev/nand0p6
/dev/nand0p7
/dev/nand0p8
/dev/nand0p9

Check partition sizes:

cat /proc/partitions

Observed map:

nand0p1 — env — 256 KB
nand0p2 — kernel1 — 6 MB
nand0p3 — rootfs1 — 36 MB
nand0p4 — kernel2 — 6 MB
nand0p5 — rootfs2 — 36 MB
nand0p6 — misc — 256 KB
nand0p7 — private — 256 KB
nand0p8 — crashlog — 256 KB
nand0p9 — UDISK — about 26 MB

Check mounts:

mount

Observed active rootfs:

/dev/root on / type squashfs (ro,relatime)

This means `/usr/share/...` is read-only and should not be patched directly.

## Locate original sound packs

Search for audio files:

find / -iname "*.wav" 2>/dev/null

find / -iname "*.opus" 2>/dev/null

Original sound packs found at:

/usr/share/sound-vendor/AiNiRobot

/usr/share/sound-vendor/XiaoMi

/usr/share/sound-vendor/XiaoMi_M88

Example files:

/usr/share/sound-vendor/AiNiRobot/first_voice.opus

/usr/share/sound-vendor/AiNiRobot/wakeup_wozai.wav

/usr/share/sound-vendor/AiNiRobot/bluetooth_connect.opus

/usr/share/sound-vendor/AiNiRobot/reset.opus

/usr/share/sound-vendor/AiNiRobot/reset_wait.opus

List active vendor directory:

ls -lah /usr/share/sound-vendor/AiNiRobot

## Mount writable UDISK

Create mount point:

mkdir -p /tmp/udisk

Mount UDISK:

mount -o rw /dev/nand0p9 /tmp/udisk

Expected kernel output may include:

EXT4-fs (nand0p9): recovery complete

EXT4-fs (nand0p9): mounted filesystem with ordered data mode

Check contents:

ls -la /tmp/udisk

Important discovery:

/tmp/udisk/sound -> /usr/share/sound-vendor/AiNiRobot

This symlink defines the active sound pack.

Check:

ls -la /tmp/udisk/sound

Expected original state:

/tmp/udisk/sound -> /usr/share/sound-vendor/AiNiRobot

## Create custom sound pack

Custom system melody files were generated with the same filenames as the original `AiNiRobot` active pack.

Target directory on device:

/tmp/udisk/sound_custom

The custom pack must contain flat files directly inside `sound_custom`, for example:

/tmp/udisk/sound_custom/first_voice.opus

/tmp/udisk/sound_custom/wakeup_wozai.wav

/tmp/udisk/sound_custom/bluetooth_connect.opus

/tmp/udisk/sound_custom/reset.opus

Not:

/tmp/udisk/sound_custom/AiNiRobot/first_voice.opus

## Upload custom files through UART

Network was not available in emergency `init=/bin/sh` mode, so upload was done through UART using a shell script.

First verify that BusyBox `printf` supports hex byte output:

printf '\x41\x42\x43' > /tmp/printf_test.bin

hexdump -C /tmp/printf_test.bin

Expected:

41 42 43 |ABC|

Then send upload script over UART.

On the device:

cat > /tmp/udisk/l07a_upload_system_sounds_AiNiRobot.sh

In Tera Term:

File -> Send file...

Send the shell script as normal text, not XMODEM/ZMODEM.

Recommended Tera Term send settings:

Bulk read: enabled
Binary: disabled
Delay per line: 5–10 ms

After transfer finishes, press Ctrl+D to finish `cat`.

Check uploaded script:

ls -lh /tmp/udisk/l07a_upload_system_sounds_AiNiRobot.sh

Expected size in this test:

about 436 KB

Run upload script:

sh /tmp/udisk/l07a_upload_system_sounds_AiNiRobot.sh

Expected result:

Custom sound files are written into:

/tmp/udisk/sound_custom

Check files:

ls -la /tmp/udisk/sound_custom | head

ls -la /tmp/udisk/sound_custom/first_voice.opus

ls -la /tmp/udisk/sound_custom/wakeup_wozai.wav

## Switch active sound pack

Backup original symlink:

mv /tmp/udisk/sound /tmp/udisk/sound.factory

Create new symlink:

ln -s /tmp/udisk/sound_custom /tmp/udisk/sound

Flush changes:

sync

Verify:

ls -la /tmp/udisk/sound

readlink /tmp/udisk/sound

Expected:

/tmp/udisk/sound -> /tmp/udisk/sound_custom

Check access through symlink:

ls -la /tmp/udisk/sound/first_voice.opus

ls -la /tmp/udisk/sound/wakeup_wozai.wav

## Reboot and test

Reboot:

reboot

If reboot does not work from emergency shell, power-cycle the device.

Expected result:

* Chinese startup voice is replaced or removed.
* Wakeup prompt is replaced by a short system melody.
* Bluetooth / reset / Wi-Fi prompts use custom system sounds.
* Rootfs remains untouched.
* Secure boot/rootfs verification remains unaffected.

## Rollback

If custom sounds fail or are unwanted, enter root shell again and mount UDISK:

mkdir -p /tmp/udisk

mount -o rw /dev/nand0p9 /tmp/udisk

Restore original symlink:

rm /tmp/udisk/sound

mv /tmp/udisk/sound.factory /tmp/udisk/sound

sync

Verify:

ls -la /tmp/udisk/sound

Expected rollback state:

/tmp/udisk/sound -> /usr/share/sound-vendor/AiNiRobot

Then reboot or power-cycle.

## Notes

Do not write to rootfs directly. It is squashfs read-only.

Do not run `saveenv` in U-Boot unless you fully understand the consequences.

Do not erase or flash NAND partitions during this procedure.

The safe part of this method is that it only uses writable `/dev/nand0p9` UDISK and changes `/tmp/udisk/sound`, which is already a vendor sound symlink.

Original active sound symlink:

/tmp/udisk/sound -> /usr/share/sound-vendor/AiNiRobot

Custom active sound symlink:

/tmp/udisk/sound -> /tmp/udisk/sound_custom

## Result

Confirmed successful on Xiaomi AI Speaker L07A:

* UART root access works via temporary `init=/bin/sh`.
* `/dev/nand0p9` mounts as writable EXT4.
* `/tmp/udisk/sound` controls active sound pack.
* Custom sound pack can be installed without modifying signed squashfs rootfs.
* Chinese voice prompts can be replaced with neutral system melodies.

No rootfs repack. No NAND flashing. No secure boot bypass needed.
