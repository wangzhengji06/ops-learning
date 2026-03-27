## Disk Type

SSD: NVME

HDD: SDX

Virtual Disk: VDX


## Disk Management

fdisk -l 

* df - partition / file

df -h / df -T

* du - folder storage

du -h / du -s

dd - customize file for test

if source / of output / bs unit / count number / skip the part I need to ignore / seek the part I need to ignore / conv: overlap or insert?

* lsblk - device itself
lsblk -f


## What is the pattern here?
Partition: fdisk -> format: mfks -> Mount


## Partition method
Partition Method: MBR(dos) / GPT(gpt)

MBR only supports 4 partiions. sda -> sda1,sda2,sda3,sda4 etc..

Why? Becuase it is 32bit 


## Parition Practice

fdisk /dev/sda ( create n ; delete d; check p; save w)

fdisk -l /dev/sda


## Filesystem Practice
mkfs.type /dev/sda


## Mount the disk to file
mount /dev/sda /path/to/dir

umount /dev/sda or umount /path/to/dir

how to automatically mount when turned on?

/etc/fstab

source / where to mount / file system type / mount option / backup?  / self-check?

mount -a


## SWAP
permanent ban / temporary ban

How? comment /etc/fstab

swapoff -a / swapon -a

echo 0 > /proc/sys/vm/swappiness 


## RAID mechanism
RAID 0  equally split between two disks 100%

RAID 1  fully copied backup for one disk 50%

RAID 5  when one broke, other can fully make up 75%


## LVM

7 physcial disks -> 7 pv -> 1 vg -> lv

Running out of storage? Add disks, add pv, enlarge vg, increase lv

Dont need this much? Reduce lv, reduce vg, remove pv then you can remove physical disks

lv folows same procedure -> partition / format / mount


## LVM commands

pvs, vgs, lvs, lvscan

pvcreate, vgcreate, lvcreate

pvremove, vgremove, lvremove 

pvdisplay, vgdisplay, lvdisplay

lvscan / lvdisplay + lvpath

lv paths: /dev/vg name/lv name

also there is pvdislay, vgdisplay etc...

## LVM procedure

1. pvcreate /dev/sda1 /dev/sdb1 /dev/sdc1

2. vgcreate -s 16M testvg /dev/sda1 /dev/sdb1   Make the smallest size 16m

3. lvcreate -l 100 -n lv1 testvg  -> Create 100 * 16M as lv1 




