# jaseytube1000.github.io
Arch Linux Installation Guide (GNOME Boxes)

(When installing Arch on GNOME Boxes, I encountered a problem where it kept looping in the beginning, but I was able to solve it by going into the EFI firmware settings to disable Secure Boot so that it would be able to choose the proper boot device. I did this by going into Device Manager to edit the "Attempt to Boot" setting.)
1. Partitioning the Virtual Drive

We then use the cfdisk command to create partitions for the virtual drive (/dev/vda) by typing:
Bash

cfdisk /dev/vda

Choose gpt and select the free space. Press Enter to edit the storage amount, which should be 2G (2 GB), and set the partition type to EFI. Leave the rest of the free space for the root partition by hovering over the free space, creating it by pressing Enter, and leaving the default size. Then hover over Quit and press Enter.

(I encountered a problem because when I tried to use the regular mkfs.fat ext4 command on the second partition, it was not working. I had unfortunately forgotten to write both partitions to the disk. To fix this, I made sure to write to the disk for each partition. I re-edited it by running the cfdisk command again, changed the size, then hovered over Write, pressed Enter, and typed 'yes'.)
2. Formatting and Mounting the Partitions

We will then make /dev/vda1 use the FAT32 format so the motherboard can read it (because it just shows where the files are):
Bash

mkfs.fat -F32 /dev/vda1

After that, format the root partition where it will use the advanced ext4 formatter to organize files properly compared to FAT32:
Bash

mkfs.ext4 /dev/vda2

To make it accessible to both the filesystem and the hardware, we will mount the root partition. The /mnt directory will temporarily act as a bridge for the boot folder to hold the GRUB files so the partition will permanently hold them later. This applies to the rest of the partitions too, and the operating system in general:
Bash

mount /dev/vda2 /mnt

To mount the EFI partition, this command will create a folder for the UEFI firmware to find at the top of the boot filesystem temporarily:
Bash

mount --mkdir /dev/vda1 /mnt/boot

To see where the partitions are mounted and verify that vda2 is mounted at /mnt and vda1 is mounted at /mnt/boot, run:
Bash

lsblk

3. Installing the Base System

Now we will use the pacstrap command to download and install the operating system and its packages:
Bash

pacstrap -K /mnt base linux linux-firmware networkmanager nano sudo openssh xfce4 sddm xorg zsh grml-zsh-config grub efibootmgr

base: Provides the basic GNU utilities that include both systemd and pacman (a package manager).linux: The Linux kernel is there as a software engine that translates for both the hardware and software.linux-firmware: Run so that it can install critical proprietary hardware drivers required for things like Wi-Fi cards, Bluetooth modules, and audio controllers.nano: Installed as a text editor for configuration files.networkmanager: Connects the system to the internet.grub: The bootloader program that loads the Linux kernel into memory using a menu.efibootmgr: Important for writing boot entries into the UEFI's firmware.xfce4: The desktop environment we are using because it is minimalistic.xorg: The server that will be the way to render graphics.zsh & grml-zsh-config: The shell we are using that will translate the things we are doing into actions.sddm: Will be the login menu visual display.-K: Important so that it will copy the cryptographic keys from the live environment to the hard drive, allowing your system to install packages signed by developers and ensuring it will not install if it has been tampered with.
4. Generating the Filesystem Table and Chrooting

Use genfstab to generate the filesystem table with -U so that it has a Universal Unique Identifier and looks at the /mnt folder for the configuration where it will save to /mnt/etc/fstab (the exact file on your permanent virtual hard drive where this map is appended and saved):
Bash

genfstab -U /mnt >> /mnt/etc/fstab

Move the pointer for the root directory through the /mnt folder to the permanent drive using the arch-chroot script to connect hardware from the live environment to the permanent drive:
Bash

arch-chroot /mnt

(Unfortunately, I was not able to do this initially because it said something did not exist, so I restarted the process and it worked.)
5. System Configuration (Time, Language, and Hostname)

We will then configure the system language and time. First, to make the system align with Chicago time, we will run the following command (ln links files with -s creating a symbolic shortcut, and -f forces the local time file to be overwritten with the timezone you want):
Bash

ln -sf /usr/share/zoneinfo/America/Chicago /etc/localtime

Then, to sync the hardware clock with the software time:
Bash

hwclock --systohc

To add American English into the list of languages you can use, append the formatted string for the language pack into the list of languages:
Bash

echo "en_US.UTF-8 UTF-8" >> /etc/locale.gen

Compile the American English packs so the system will display the text:
Bash

locale-gen

Display the system language and overwrite the current language configuration to replace it with English:
Bash

echo "LANG=en_US.UTF-8" > /etc/locale.conf

Display and force the system to adopt the name arch-vm:
Bash

echo "arch-vm" > /etc/hostname

Change the password for the administrator with the passwd command:
Bash

passwd

6. Installing the Bootloader

Install the bootloader software:
Bash

pacman -S grub efibootmgr

(Pacman is the package manager for Arch, and -S installs packages. GRUB installs the bootloader, and efibootmgr is super important as it translates GRUB actions to the hardware.)

Copy the boot files into the drive, format it to 64-bit (--target=x86_64-efi), show where GRUB is mounted (--efi-directory=/boot), and give it a name (--bootloader-id=GRUB):
Bash

grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB

Scan for the Linux kernels and make the code to load them, outputting (-o) that code to the main file:
Bash

grub-mkconfig -o /boot/grub/grub.cfg

7. Creating Users and Setting Permissions

We will then add users with their main directory through -m, use -G wheel for a background administrator group, and -s for the shell:
Bash

useradd -m -G wheel -s /bin/zsh jeff
passwd jeff

useradd -m -G wheel -s /bin/zsh codi
passwd codi

Go into nano to give sudo privileges to everyone in the wheel group by deleting the # in front of the wheel group line:
Bash

EDITOR=nano visudo

(Okay, another difficulty: I did not know the wheel group was strict, so right now it looks okay above, but originally it had a space.)

Enable color output by uncommenting it out in the pacman configuration file:
Bash

nano /etc/pacman.conf

8. Enabling Background Services

Enable important background services with the systemctl command:
Bash

systemctl enable sddm
systemctl enable NetworkManager
systemctl enable sshd

9. ZSH Aliases and Theming

Copy the default ZSH configuration into Jeff's account:
Bash

cp /etc/zsh/zshrc /home/jeff/.zshrc

Add aliases and colors for both accounts:
Bash

# Full package upgrade
echo "alias update='sudo pacman -Syu'" >> /home/jeff/.zshrc 

# Permanent color on switch
echo "alias ls='ls --color=auto'" >> /home/jeff/.zshrc 

# Allows for color customization and enables bold text
echo 'export LS_COLORS="ex=1;32:di=1;31:fi=0;33"' >> /home/jeff/.zshrc

# Shows available disk space
echo "alias df='df -h'" >> /home/codi/.zshrc 

# Copies these same aliases into Codi's account
cp /home/jeff/.zshrc /home/codi/.zshrc

# Transfers ownership
chown codi:codi /home/codi/.zshrc

10. Finalizing

Finally, exit out and power off the virtual machine. Make sure to remove the ISO image from GNOME Boxes:
Bash

exit
poweroff

Then start the machine and select GRUB EFI BootNext to get into the environment.
