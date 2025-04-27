# MacOS Development Setup

## Orbstack

We will be using an orbstack machine to do all out linux development work.

Install orbstack

```sh
brew install orbstack
```

Open orbstack, under `Machine` add a new machine. I am using `Ubuntu 24.10 Oracular Oriole`.

Open a terminal and enter the following command

```sh
orb
```

You have now ssh'd into orbstack machine (exit with `Ctrl+D`). You can verify with `uname -a`.

## Install essential tools

Install some linux images so that libguestfs tools can work since they don't ship with an appliance.

```sh
sudo apt install linux-image-generic
```

We'll need to make the installed kernel artifact readable by libguestfs tools.

```sh
sudo chmod -R o+r /boot
```

Install guestfs-tools and cloud-utils

```sh
sudo apt install guestfs-tools cloud-utils
```

## Ubuntu minimal cloud image

We'll grab a minimal Ubuntu images (both `arm64` and `x86_64/amd64`) for our test VMs.

```sh
wget https://cloud-images.ubuntu.com/minimal/releases/oracular/release/ubuntu-24.10-minimal-cloudimg-arm64.img
```

```sh
wget https://cloud-images.ubuntu.com/minimal/releases/oracular/release/ubuntu-24.10-minimal-cloudimg-amd64.img
```

## Qemu ARM64 setup

Install `qemu`

```sh
sudo apt install qemu-system
```

Create an empty qcow2 rootfs. We create our own rootfs so that we have sufficient space to install extra packages; or do development and testing if needed.

```sh
qemu-img create -f qcow2 -o preallocation=metadata ubuntu_arm64_rootfs.qcow2 20G
```

Inspect minimal cloud image

```sh
virt-filesystems --long -h --all -a ubuntu-24.10-minimal-cloudimg-arm64.img
```

Output:
```sh
Name        Type        VFS   Label            MBR  Size  Parent
/dev/sda1   filesystem  ext4  cloudimg-rootfs  -    2.4G  -
/dev/sda13  filesystem  ext4  BOOT             -    890M  -
/dev/sda15  filesystem  vfat  UEFI             -    97M   -
/dev/sda1   partition   -     -                -    2.5G  /dev/sda
/dev/sda13  partition   -     -                -    923M  /dev/sda
/dev/sda15  partition   -     -                -    99M   /dev/sda
/dev/sda    device      -     -                -    3.5G  -
```

From the output `/dev/sda1` of `ext4` type is the rootfs.

Expand the minimal cloud image rootfs onto our rootfs.

```sh
virt-resize --expand /dev/sda1 ubuntu-24.10-minimal-cloudimg-arm64.img ubuntu_arm64_rootfs.qcow2
```

Reinstall GRUB (TODO: I am not sure why this step is needed)

```sh
virt-customize -a ubuntu_arm64_rootfs.qcow2 --run-command 'grub-install /dev/sda'
```

## SSH Setup

Generate a ssh key that we'll use to ssh into the qemu instance

```sh
ssh-keygen -t rsa -b 4096
```

Ubuntu cloud images do not have default login/password. We need to provide hostname and ssh-keys for login. For this create `metadata.yaml` and `user-data.yaml` as shown below:

```sh
$ cat metadata.yaml
instance-id: iid-local01
local-hostname: qemu-arm64
```

```sh
$ cat user-data.yaml
#cloud-config
ssh_authorized_keys:
```

Add your public rsa key to the list of authorized keys.

```sh
echo " - $(cat ~/.ssh/id_rsa.pub)" >> userdata.yaml
```

Now that we have these files in place, create `seed.img`

```sh
cloud-localds seed.img user-data.yaml metadata.yaml
```

## Get the Linux kernel source

```sh
sudo apt install git
git clone https://github.com/torvalds/linux.git
```

## Build the kernel for Qemu

```sh
sudo apt update
sudo apt install build-essential llvm clang lld
```

Create a configuration suitable for qemu starting from arm64 defconfig.
```sh
make -j`nproc` LLVM=1 LLVM_IAS=1 CC=clang ARCH=arm64 defconfig

./scripts/config --file .config \
    -e VIRTIO_BLK \
    -e VIRTIO_NET \
    -e HW_RANDOM_VIRTIO \
    -e AUTOFS4_FS \
    -e NET_FAILOVER \
    -e VIRTIO_PCI_LIB \
    -e FAILOVER \
    -e DEVTMPFS \
    -e DEVTMPFS_MOUNT \
    -e VFIO_PLATFORM \
    -e ISO9660_FS

make -j`nproc` LLVM=1 LLVM_IAS=1 CC=clang ARCH=arm64 olddefconfig
```

Build it

```sh
sudo apt install flex bison bc libssl-dev
```

```sh
make -j`nproc` LLVM=1 LLVM_IAS=1 CC=clang ARCH=arm64
```

The built `amr64 Image` will be under `linux/arch/arm64/boot/Image`.

## Booting the Image

Check your rootfs image to identify the rootfs partition.

```sh
virt-df -h -a ubuntu_arm64_rootfs.qcow2
```

Output

```sh
Filesystem                                Size       Used  Available  Use%
ubuntu_arm64_rootfs.qcow2:/dev/sda1        97M       6.3M        91M    7%
ubuntu_arm64_rootfs.qcow2:/dev/sda2       890M        33M       795M    4%
ubuntu_arm64_rootfs.qcow2:/dev/sda3        18G       541M        18G    3%
```

Here the rootfs partition is `/dev/sda3`, when we boot the image in qemu before the `/dev/sda3` will be specified as `/dev/vda3`.


```sh
qemu-system-aarch64 \
    -M virt \
    -accel tcg \
    -cpu cortex-a72 \
    -m 8G \
    -smp 4 \
    -nographic \
    -drive file=ubuntu_arm64_rootfs.qcow2,if=none,id=hd0,format=qcow2 \
    -device virtio-blk-device,drive=hd0 \
    -kernel <linux>/arch/arm64/boot/Image \
    -append "console=ttyAMA0 root=/dev/vda3 rw"
```

Note: that we aren't able to use KVM since mac doesn't expose this capabillity to our Orbstack VM so `/dev/kvm` isn't available. We fallback to `tcg` acceleration.