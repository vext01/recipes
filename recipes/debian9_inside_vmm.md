# Running Debian 9 inside OpenBSD `vmm(4)`

At the time of writing, `vmm(4)` does not emulate CDROM after initial boot.
This means you cannot install Debian directly under vmm(4).

We can work around this by doing the initial install under qemu.


## Process

### Make a disk image with `vmctl(8)`.

```
$ vmctl create deb9.img -s 5G
```

### Boot the Debian install CD under qemu with the disk image attached.

```
$ qemu-system-x86_64 -cdrom debian-9.3.0-amd64-xfce-CD-1.iso -hda deb9.img -boot d -m 3G -net nic -net user
```

### Install Debian as usual.

This will be slow, as qemu is not accelerated on OpenBSD.

Once the install is done, shut down the VM.

### Boot the new system under qemu, but without the install CDROM inserted.

```
$ qemu-system-x86_64 -hda deb9.img -m 3G -net nic -net user
```

### Change the console to the serial line.

At the time of writing, vmm(4) doesn't do VGA emulation, so we have to use the
serial line as a console.

Login in and edit `/etc/default/grub`, adding the following `console=ttyS0` to
`GRUB_CMDLINE_LINUX_DEFAULT`. You should then have a line like:

```
GRUB_CMDLINE_LINUX_DEFAULT="quiet console=ttyS0"
```

Install the new boot-loader config to the disk, then halt the VM.

```
# update-grub
...
# halt -p
```

### Boot the VM under vmm(4).

```
$ doas vmctl start deb9 -d deb9.img -i 1 -L -m 3G -c
```

### Update the networking configuration.

vmm(4) emulates a different kind of network device to qemu, so we have to
change the config.

Edit `/etc/networks/interfaces` and replace all instances of `ens3` with
`enp0s3`. You should then have a section like this:

```
# The primary network interface
allow-hotplug enp0s3
iface enp0s3 inet dhcp<Paste>
```

### Grant the VM network access.

On the OpenBSD host, edit `/etc/pf.conf` and add a lines like this:

```
vm_ext=iwm0
vm_dns=8.8.8.8
...
pass out on $vm_ext from 100.64.0.0/10 to any nat-to ($vm_ext)
pass in proto udp from 100.64.0.0/10 to any port domain \
     rdr-to $vm_dns port domain
```

Don't forget that the order of `pf(4)` rules matters!

Apply the new rules:

```
# pfctl -f /etc/pf.conf`
```

Change `vm_ext` and `vm_dns` to: an interface through which to grant the VM
internet access, and a valid DNS server respectively.

Then enable ipv4 (or ipv6, if you need it) forwarding:

```
# sysctl net.inet.ip.forwarding=1
# echo "net.inet.ip.forwarding=1" >> /etc/sysctl.conf
```

### Reboot the VM.

And you are done.

It's probably a good idea to copy the image somewhere safe so you can use as a
template for Linux VMs.
