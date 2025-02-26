---
title: "Tomoyo LSM installation"
date: "2025-02-25"
ShowToc: true
---

## Installing TOMOYO on Ubuntu 24.04.01

First we check if kernel has TOMOYO enabled:

```bash
grep tomoyo_write_inet_network /proc/kallsyms
```

It should output something like this:

```bash
0000000000000000 T tomoyo_write_inet_network
```

Outputing mean tomoyo LSM is compiled into the kernel, we do not have to activate it in the kernel which makes our job easier.

## Installing dependencies

We need to install some dependencies before we can download and compile TOMOYO userspace tools.

```bash
sudo apt-get -y install wget patch gcc make libncurses-dev
```

## Configuring kernel

We can skip this part because our kernel already has TOMOYO enabled. If not we would have to download kernel sources and configure it. This part is covered in [TOMOYO documentation](https://tomoyo.sourceforge.net/2.6/chapter-3.html.en#3.1.3).

## Installing userspace tools

First we download tools from sourceforge and verify the signature with kumaneko-key.
We can then extract the archive and compile the tools.

```bash
wget https://sourceforge.net/projects/tomoyo/files/tomoyo-tools/2.6/tomoyo-tools-2.6.1-20241111.tar.gz
wget https://sourceforge.net/projects/tomoyo/files/tomoyo-tools/2.6/tomoyo-tools-2.6.1-20241111.tar.gz.asc
wget https://tomoyo.sourceforge.net/kumaneko-key
gpg --import kumaneko-key
gpg --verify tomoyo-tools-2.6.1-20241111.tar.gz.asc
tar -zxf tomoyo-tools-2.6.1-20241111.tar.gz
cd tomoyo-tools
make -s USRLIBDIR=/usr/lib
sudo make -s USRLIBDIR=/usr/lib install
```

## Initialize configuration

Here we add tomoyo userspace tools to our PATH:

```bash
export PATH=$PATH:/usr/sbin
```

Not certain but I added this to my `.bashrc` just in case.

Next we initialized policy:

```bash
sudo /usr/lib/tomoyo/init_policy
```

We should get bunch of OKs.

## Configure bootloader

```bash
sudo nano /etc/default/grub
```

Here we edit line `GRUB_CMDLINE_LINUX` to include `security=tomoyo`.

```bash
GRUB_CMDLINE_LINUX="quiet splash security=tomoyo"
```

Then we update grub.

```bash
sudo update-grub
```

## Reboot

Now we reboot the system:

```bash
sudo reboot
```

After reboot we can check if TOMOYO is enabled. Using this command:

```bash
cat /sys/kernel/security/lsm
```

Output should look something like this:

```bash
lsm=capability,yama,loadpin,safesetid,selinux,tomoyo
```

We have TOMOYO enabled and running. We can now start configuring it.
