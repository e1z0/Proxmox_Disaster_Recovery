# 🛠  When Disaster Recovery Becomes Necessary

There have been times in my life when certain "stubborn" servers — particularly models like **HP ProLiant** — refuse to boot from disks located behind RAID controllers, when configured in **JBOD** mode. In such cases, I'm often forced to rely on alternative boot devices such as an **internal SD card slot** or an **internal USB** port to hold a bootable OS.

However, these solutions are far from ideal. SD cards and USB flash drives are not designed for high write endurance and may fail unexpectedly. When they do — and especially if the SD card slot or USB port itself becomes damaged — we’re left with a **completely unbootable server**. This quickly escalates into a **critical disaster scenario**.

There are typically three fallback options for booting such a server:

- Use an **external USB port** (unreliable, especially in rack environments).
- Boot over **PXE (network boot)**.
- Use **virtual media** via iLO (e.g., a virtual CD-ROM mounted through a web interface).

A potential fourth option exists if the motherboard has a free **internal SATA port**, but for the sake of this article, we’ll assume it doesn’t.

# 🧯 Objectives of This Article

This write-up is intended to guide you through **restoring functionality** to such systems **as quickly as possible**, with a focus on disaster recovery scenarios. For servers like HP ProLiant, we'll explore how to leverage **iLO features** such as PXE boot and Virtual Media.

The article is structured into three main parts:

1. **Preparing for Disaster** – Prevention is better than cure. We’ll start by discussing how to properly prepare your server so that when failure eventually happens, you’re ready.
2. **Recovery After Disaster** – If your system is already down and unbootable, we’ll walk through steps to enter recovery mode, access critical files (like kernel and initrd), and rebuild a bootable environment.
3. **Boot Strategies** – Finally, we’ll cover how to boot the system using PXE and Virtual CD-ROM when all else fails.

# 🛡  Preparing for Disaster

⚠️  If your system is already unbootable, please skip ahead to the next section: **Recovery After Disaster**.

```
apt-get install grub-pc-bin xorriso grub-common mtools
git clone https://github.com/e1z0/Proxmox_Disaster_Recovery.git
cd Proxmox_Disaster_Recovery
cp /boot/vmlinuz-$(uname -r) ISO/boot/vmlinuz
cp /boot/initrd.img-$(uname -r) ISO/boot/initrd
cp /boot/vmlinuz-$(uname -r) PXE/boot/vmlinuz
cp /boot/initrd.img-$(uname -r) PXE/boot/initrd
grub-mkrescue -o proxmox-rescue.iso ~/ISO
```

You now have:

* A **rescue ISO** (for mounting as virtual media in iLO or similar IPMI)
* A **PXE folder** (for booting via TFTP/Netboot)

# 💀 Recovery After Disaster

If your Proxmox server is unbootable and the internal boot medium is dead:

- Download **[SystemRescueDisk](https://www.system-rescue.org/Download/)**
- **Boot** the server with **SystemRescueDisk.iso** via iLO virtual CD-ROM
- At the bootloader menu, press **Tab** and append kernel parameters: `nofirewall rootpass=1234`
- Once booted, connect via SSH using password 1234 (Because using iLO remote control is... a disaster of its own.)
- Mount your Proxmox root filesystem: `mount /dev/mapper/pve-root /mnt`
- Transfer latest working kernel and initrd files from /boot to another Linux machine via sftp
- On that machine, follow the steps in **Preparing for Disaster** to build the recovery **ISO** or **PXE** boot environment.

## 🔧 Optional: Chroot into the Installed System

If you need to make repairs or changes within your existing Proxmox installation:

```
mount /dev/sdg1 /mnt/boot   
mount --rbind /dev/ /mnt/dev
mount --bind /proc /mnt/proc
mount --rbind /sys/ /mnt/sys
mount –-rbind /run/ /mnt/run
chroot /mnt bash
```

You now have access to your live system and can modify configs, reinstall bootloaders, or regenerate initrd.

# 🚀 Boot Strategies

## PXE Boot

You can boot the server using PXE via either:

* tftpd64 on Windows
* dnsmasq on Linux


Here is a sample dnsmasq configuration:
```
port=0
interface=eth0
dhcp-range=10.2.14.0,proxy
dhcp-boot=pxelinux.0
enable-tftp
tftp-root=/srv/tftp
```

Then, copy everything from your PXE/ folder into /srv/tftp.

💡 Important: You must create a config file that matches your server’s MAC address (formatted for PXELINUX).

For example, if your server's MAC is bc:24:11:89:23:a8, then your config file must be:

```
/srv/tftp/pxelinux.cfg/01-bc-24-11-89-23-a8
```
(PXELINUX uses prefix 01- and replaces : with -.)


## Virtual Media

To use the rescue ISO via virtual media:

- Open your server's iLO interface
- Mount proxmox-rescue.iso as a virtual CD/DVD drive
- You can host the ISO on your local machine using a simple web server:
```
python3 -m http.server
```
Ensure the web server is in the same LAN as your iLO interface for faster performance.

