# MiniLinux-From-Scratch
A simple tutorial to create a small, self-contained Linux distribution derived from Alpine Linux. The approach and scripts are intentionally generic so the same methodology can be adapted to other base distributions. 

---

## Requirements

- **A compiled Linux kernel**: This is the core component of any Linux system, responsible for managing hardware and system resources.
- **Docker**

---

## Linux kernel compilation

- Download a Linux kernel archive [here](https://www.kernel.org/).
- Compile it with :

```
$ make defconfig
$ make
```

This process generates the kernel binary using the default configuration settings.

---

## Disk image

**Create an empty 450 MB file to prepare a disk image for the system** :

```
$ truncate -s 450M disk.img
```

This file acts as a placeholder for an entire hard disk. A BIOS partition table is created in it :

```
$ /sbin/parted -s ./disk.img mktable msdos
```

**Add a primary partition of type ext4 occupying all the space** :

```
$ /sbin/parted -s ./disk.img mkpart primary ext4 1 "100%"
```

This command specifies how to divide the disk image into usable partitions.

**Flag it as bootable** :

```
$ /sbin/parted -s ./disk.img set 1 boot on
```

This allows the BIOS to recognize the partition as bootable.

**Create a loopback device to access the image and its partitions** :

```
$ sudo losetup -Pf disk.img
```
A loopback device allows for the use of a file as a block device.

**Note which device is associated with `disk.img`** :

```
$ losetup -l
```

This confirms the loopback device's assignment.

**Format the first partition in ext4** (X is your previously created loopback device) :

```
$ sudo mkfs.ext4 /dev/loopXp1
```

Formatting prepares the partition for use by creating a file system.

**Create a working directory and mount the partition** :

```
$ mkdir -p /tmp/my-rootfs
$ sudo mount /dev/loopXp1 /tmp/my-rootfs
```

Mounting makes the partition accessible for read and write operations.

**Install a minimal Alpine Linux via Docker in our working directory** :

```
$ docker run -it --rm -v /tmp/my-rootfs:/my-rootfs alpine
```

This pulls a lightweight Alpine container image and mounts it to your working directory.

## Image population under Docker

**The following steps are performed in Docker after the `docker run` command. We configure Alpine with a minimal system, basic utilities, and a C compiler** :

```
$ apk add openrc util-linux build-base clang llvm bpftool libbpf-dev linux-headers
```

These commands install essential packages for system initialization and development.

**Configure a terminal for serial access via QEMU** :

```
$ ln -s agetty /etc/init.d/agetty.ttyS0
$ echo ttyS0 > /etc/securetty
$ rc-update add agetty.ttyS0 default
$ rc-update add root default
```

**Define a password for root** :

```
$ passwd root
```

**Activate pseudo file systems at startup** :

```
$ rc-update add devfs boot
$ rc-update add procfs boot
$ rc-update add sysfs boot
```

These commands enable virtual filesystems that provide system information and device management.

**Copy the Docker configuration to the mounted partition** :

```
$ for d in bin etc lib root sbin usr; do tar c "/$d" | tar x -C /my-rootfs; done
```

This step transfers essential directories from the container to the new filesystem.

**Create special directories** : 

```
$ for dir in dev proc run sys var; do mkdir /my-rootfs/${dir}; done
```

**Exit Docker** : 

```
$ exit
```

---

## Installing GRUB in the image

**You can edit Alpine's welcome message** :

```
$ sudo vim /tmp/my-rootfs/etc/issue
```

**You must have a compiled Linux kernel to install in the partition. Create the boot directory for GRUB and the kernel** :

```
$ sudo mkdir -p /tmp/my-rootfs/boot/grub
```

This is necessary for GRUB to locate and boot the kernel.

**Copy the compiled kernel** (from the directory `linux` of the choosen version) :

```
$ sudo cp linux/arch/x86/boot/bzImage /tmp/my-rootfs/boot/vmlinuz
```

**Create and edit `grub.cfg`** :

```
$ sudo vim /tmp/my-rootfs/boot/grub/grub.cfg
```

This configuration file dictates how GRUB will boot your system.

```
serial
terminal_input serial
terminal_output serial
set root=(hd0,1)
menuentry "MiniLinux-From-Scratch" {
  linux /boot/vmlinuz root=/dev/sda1 console=ttyS0
}
```

This setup allows GRUB to load the kernel at startup.

**Install GRUB via loopback for BIOS boot** :

```
$ sudo grub-install --directory=/usr/lib/grub/i386-pc --boot-directory=/tmp/my-rootfs/boot /dev/loopX
```

This command installs the GRUB bootloader onto the loopback device, enabling the system to boot within QEMU.

---

## Unmount & Launch

**Unmount the partition and detach the loopback** :

```
$ sudo umount /tmp/my-rootfs
$ sudo losetup -d /dev/loopX
```

These commands clean up the environment by unmounting the filesystem and releasing the loopback device.

**Launch QEMU on disk image** : 

```
$ qemu-system-x86_64 -hda disk.img -nographic
```

This command starts the QEMU emulator to run your newly created Linux distribution without a graphical interface.
