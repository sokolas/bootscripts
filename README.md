# bootscripts
UEFI Linux boot scripts and instructions

from https://github.com/slytomcat/UEFI-Boot/wiki
### Combine ESP and boot in one partition. ###
Instead of repetitive copying/removing of kernels and initrds to/from EFS let's force the system to perform all such action over kernels/initrds directly within the ESP partition. Let's make /boot partition on ESP. 
This idea is developed in the UEFI-Boot project.
> The only one small disadvantage of keeping kernels on FAT is that FAT doesn't support links and re-installation of kernel may fail because of this. If you run into such problem just remove kernel and install it back (instead of re-installation).

## UEFI-Boot solution ##
> UEFI-Boot primary designed for Ubuntu distribution. Note that some distribution related things have to be changed to implement this solution on other Linux distributions.  

First we need to move all kernel's/ staff to the root of ESP partition.

`# mv /boot/*-generic* /boot/efi/`

Dismount ESP partition.

`# umount /boot/efi`

Clear /boot - it will be just a mount point for ESP partition. 

`# rm -rf /boot/*`

Next, we need to change the mount point of ESP partition in /etc/fstab

`# sed -i 's/\/boot\/efi/\/boot/' /etc/fstab`

Finally, mount ESP into /boot

`# mount /boot`

Now all updates of kernels and initrds will be performed directly within the ESP file system, and the in same time the UEFI has ability to access kernel and initrd without any additional drivers.

(from https://wiki.archlinux.org/index.php/systemd-boot)
Installing the EFI boot manager

To install the systemd-boot EFI boot manager, first make sure the system has booted in UEFI mode and that UEFI variables are accessible. This can be checked by running the command `efivar --list.`

It should be noted that systemd-boot is only able to load the EFISTUB kernel from the EFI system partition (ESP). To keep the kernel updated, it is simpler and therefore recommended to mount the ESP to /boot. If the ESP is not mounted to /boot, the kernel and initramfs files must be copied onto that ESP. See EFI system partition#Alternative mount points for details.

*esp* will be used throughout this page to denote the ESP mountpoint, i.e. `/boot`.

With the ESP mounted to esp, use bootctl(1) to install systemd-boot into the EFI system partition by running:

`# bootctl --path=esp install`

This will copy the systemd-boot boot loader to the EFI partition: on a x64 architecture system the two identical binaries *esp*/EFI/systemd/systemd-bootx64.efi and *esp*/EFI/BOOT/BOOTX64.EFI will be transferred to the ESP. It will then set systemd-boot as the default EFI application (default boot entry) loaded by the EFI Boot Manager. 

The update utility itself has a rather simple algorithm: first it removes all old Ubuntu* boot options from systemd uefi loader, then it creates back new boot options for all installed kernels. BASH script of utility is located to in /usr/bin/uefiboot-update.

The utility has configuration file /etc/uefiboot.conf. It provides ability to specify the root partition reference, kernel boot parameters, and kernel suffix (suffix is required for signed kernels in case of SecureBoot activation). When config parameters are not set then utility:
- takes the root reference and subvolume name/id (for btrfs) from the /etc/fstab file,
- set kernel boot parameters to 'ro quiet' and kernel suffix as empty string.
Another way to change uefiboot-update behavior - pass necessary parameters via command line options (see readme file for details)      

The last question that we have to solve - to update UEFI boot options every time when a new kernel comes with update or when an old kernel is removed. It can be easily organized via kernel triggers (the same way as the GRUB update script is initialized).
Triggering of utility is organized via links in /etc/kernel/postinst.d and /etc/kernel/postrm.d.

`# ln -s /usr/bin/uefiboot-update /etc/kernel/postinst.d/uefiboot-update`

`# ln -s /usr/bin/uefiboot-update /etc/kernel/postrm.d/uefiboot-update`

As we excluded Shim and GRUB from the booting chain, now we can (and have to) remove them from the system. os-prober (a component of GRUB that searches for other OS installations) is also not required any more. 

`# apt-get purge grub* shim os-prober`

Additionally we have to instruct the system do not install recommended package as kernel has GRUB in recommends.

`# echo 'APT::Install-Recommends "false";' > /etc/apt/apt.conf.d/zz-no-recommends`

Finally, we have to trigger the utility manually to prepare UEFI for start the system next time.

`# uefiboot-update`   

Don't forget to adjust the UEFI boot timeout (the default value can be 5-10 seconds):

`# efibootmgr -t0`
