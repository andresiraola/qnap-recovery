Overview
========

This data recovery method uses a host machine running Linux to assemble the
md raid and a guest VM running the QNAP firmware to access the LVM volumes.

A running QNAP device is not necessary.

利用qemu虚拟化，在Linux PC中恢复QNAP Raid数据。

QNAP NAS硬件坏了之后要恢复数据是个很麻烦的事，官方的推荐方法是重新买一个同型号的硬件，这个成本太高(~~不会再考虑QNAP~~)。QNAP使用软件Raid方案，并对LVM做了一些修改，在普通PC上，无法用Linux正常挂载QNAP的LVM卷（还好没有改Raid）。在github上找到的这个方案使用qemu和QNAP固件虚拟QNAP软件系统，从而恢复数据。这里对原方案进行补充修改，与原方案不同之处是不考虑硬盘加密。另外，使用完整固件启动系统，而不是部分更新的固件（主要是官方找到的固件已包含完整系统）。此处仅测试了TX-651的固件，恢复4个硬盘构成的Raid5，未使用磁盘加密，其他型号和配置的数据恢复需自行参考解决。

**Tips:** 

1. 建议在恢复之前先做好硬盘数据备份。
2. 如果硬盘接口不够，买一个PCI的SATA扩展卡



Preparing the disk array
========================

* 将QNAP Raid硬盘接到Linux PC
* the disk array should be visible on the device-mapper level, either
  automatically after boot, or after mdadm -A -R --scan
* test the md block device is accessible.
  We are only interested in the data device, which should be `/dev/md1`.

  ``` bash
  mdadm --detail /dev/md1
  mdadm /dev/md1
  ```

Preparing a VM with an up-to-date firmware
==========================================

* Download FW  from qnap.com (e.g. `TS-X51_20211223-4.5.4.1892.zip`)
  and unzip
  
  ```
  unzip TS-X51_20211223-4.5.4.1892.zip
  ```
  
* decrypt the firmware

  after clone the repo, do

  ``` bash
  make TS-X51_20211223-4.5.4.1892.tgz
  ```

* extract firmware and patch initrd

```bash
mkdir firmware
tar xzf TS-X51_20211223-4.5.4.1892.tgz -C firmware
cd firmware
unlzma <initrd.boot >initrd.cpio
mkdir initrd
cd initrd
cpio -i <../initrd.cpio
patch -p1 <../../init_check.sh.diff
find . | cpio --quiet -H newc -o | lzma -9 >../initrd.lzma
cd ..
```

* build disk image from rootfs2.bz

```
dd if=/dev/zero of=usr.img bs=1k count=200k
mke2fs -F usr.img
mkdir rootfs2
cd rootfs2
tar --xz -xpf ../rootfs2.bz
mount ../usr.img home
cp -a usr/* home/
umount home
```

* create an img to save the recovered data

```sh
qemu-img create restore.img 1536G
```

* start the VM

```sh
qemu-system-x86_64 -s -kernel bzImage -nographic -initrd initrd.lzma -snapshot -drive format=raw,file=qnap-hdd.img -hdb usr.img -drive format=raw,file=restore.img -m 4G --enable-kvm
```

* login with admin / admin

* mount the usr.img

```sh
  mkdir /usr
  mount /dev/sdb /usr
```

* logout and login

* initialize lvm as it failed previosly because of the missing /usr

```sh
/sbin/daemon_mgr lvmetad start "/sbin/lvmetad"
```

* prepare the raid stuff. tested on a single disk raid

```sh
mkdir -p /run/mdadm/
mdadm --examine --scan > /etc/mdadm.conf
mdadm --assemble --scan
```

* do the pv & lv stuff

```sh
pvscan --cache /dev/md2 # pick the biggest

# just to visualize
pvs
lvs

# activate the thin pool and the volume
lvchange -a y vg2/lv2

# mount it!
mount -t ext4 /dev/mapper/vg2-lv2 /mnt/ext

# enjoy
ls /mnt/ext
```

* prepare the restore disk

```sh
# list and identify the restore disk
parted -l

# partition it
parted /dev/sdc
> print
> mklabel gpt
> mkpart primary ext4 0% 100%
> quit

# format
mke2fs -t ext4 /dev/sdc1

# mount it
mkdir -p /mnt/restore
mount -t ext4 /dev/sdc1 /mnt/restore
```

* copy the files

```sh
cp -av /mnt/ext/. /mnt/restore/
```

* use `control + a` then `x` to exit the qemu monitor
