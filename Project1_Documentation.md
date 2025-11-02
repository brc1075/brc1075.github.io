# Project 1 Documentation

This file documents my steps and challenges while ________




## Pre Installation

 1. Install Arch ISO on Windows. Choose an HTTP mirror and download the latest x86_64 file (for Windows in my case). Also download the signature file for verification.
    - I initially chose [mirror.arizona.edu](https://mirror.arizona.edu/archlinux/iso/2025.10.01/). 

 2.  Verify the ISO signature with GnuPG since the mirror site is unknown/not trusted
     - I downloaded GnuPG from [Gpg4Win](https://www.gpg4win.org/)
     - Import the key using:
  `**gpg --keyserver keyserver.ubuntu.com --recv-keys 0x3E80CA1A8B89F69CBA57D98A76A5EF9054449A5C**`
     - Verify using 
 `gpg --verify archlinux-2025.10.01-x86_64.iso.sig archlinux-2025.10.01-x86_64.iso`
 
 **NOTE:** My original ISO did not verify. I downloaded a second ISO from [us.mirrors.cicku.me](https://us.mirrors.cicku.me/archlinux/iso/2025.10.01/) . This ISO verified.
 
 3. Download Oracle VirtualBox
 4. Attach the ISO as a virtual CD/DVD in VM Settings
    - Create a new VM. Allocate 4GB memory, 4 CPU's, 30GB Disk space
    - Enable UEFI in VM Settings
    - Allocate 128Mb Video Memory
   
  **NOTE:** I originally only allocated 8GB of disk space. This later caused issues in partitioning and i had to return to this step.
  
5.   Press run to start the live environment! Will end up with shell prompt `root@archiso ~# `.  This is where installation commands are run.
6. Configuration
   - I skipped step 1.5 from the wiki since I'm already using a US keyboard layout
   - I changed the font using `setfont ter-132b`. It was way too big.
   - I changed the font again using `setfont ter-116n`. Much better!
7. Verification
   - Verify using `# cat /sys/firmware/efi/fw_platform_size`
   - This returned 64 which means we're in UEFI 64-bit (good!)
 8. Connect to Internet
    - Type `ip link` to view network devices
    - Verified connection by pinging `# ping ping.archlinux.org`
    - Data was successfully transmitted and received!
    - Exit the ping using `CTRL+C`
8. Verify Clock
   - Type `timedatectl` and verify the clock is synchronized
9. Disk Partition
   - Type `lsblk` which returned `sda 8:0 0 30G 0 disk`. This indicates the virtual drive is in /dev/sda
   - Enter fdisk using `fdisk /dev/sda`
   - Enter `g` to create a new GPT table
   - Create the EFI partition by typing `n` and accepting default settings. On the last sector setting, type `+512M` to create a 512MiB EFI partition.
   - Set it's type to EFI system by inputting `t` then inputting `1`
   - Create the second (root) partition by entering `n`. Accept all defaults
   - Enter `p` to verify output. It matched the wiki!
   - Enter `w` to write the partition table and exit fdisk.
  
**NOTE:** This step is where I had to go back to step 4 since I did not allocate enough disk space.

10. Format and Mount Partition
    - Format the first partition as FAT32 by entering `mkfs.fat -F32 /dev/sda1`
    - Format the second partition with the standard Linux filesystem by entering `mkfs.ext4 /dev/sda2`
    - Mount the main filesystem (the "root") to the temporary directory /mnt by entering `mount /dev/sda2 /mnt`
    - Mount the EFI Partition by entering `mount /dev/sda1 /mnt/boot`
    - Enter `lsblk` to verify the commands worked!
 
 ## Installation
 1. Verify/Select Mirrors - This is an important step to get right.
    - Check the current list by entering `cat /etc/pacman.d/mirrorlist`, which will show the current mirror list. Each line is a mirror server. In my case, there were multiple US servers and some European ones, so my list is satisfactory for this project. 
11. Verify the Mirror List Works  fefefe
    - Update the package database by entering `pacman -Syy` 
    - The command works without error, the mirrors work correctly!
12. Install Essential Packages
    - Enter `pacstrap -K /mnt base linux linux-firmware networkmanager nano`  
    - Base - shell, essential tools, etc
    - Linux - linux kernel
    - Linux-firmware - firmware blobs for hardware
    - Networkmanager - networking after reboot
    - Nano - text editor
    - This step took 1-2 minutes to complete.
    
 ## System Configuration  
 1. Generate fstab File
    - Tells linux to auto mount at boot
    - Enter `genfstab -U /mnt >> /mnt/etc/fstab`. You will not get an output.
    - Verify using `cat /mnt/etc/fstab`
 2. Chroot
    - Move shell into new system so further commands configure it instead of the live ISO
    - Enter `arch-chroot /mnt`
    - Stay inside this chroot for the rest of system configuration
 3. Time Configuration
     - I original entered `-sf /usr/share/zoneinfo/America/Chicago /etc/localtime and hwclock --systohc` from the wiki
     - Enter `timedatectl status` to verify time configuration is synced. The wiki command did not work for me.
     - I then tried entering `timedatectl set-timezone America/Chicago`. This worked after verifying!
  4. Localization
      - Enable a locale and set a system language
      - Enter `sed -i 's/^#\(en_US.UTF-8\)/\1/' /etc/locale.gen` to remove the # from en_US.UTF-8 UTF-8
      - Enter `locale-gen` to build locales in /etc/locale.gen
      - Enter `echo 'LANG=en_US.UTF-8' > /etc/locale.conf` to set default language for programs.
  5. Network Configuration
     - Set and map, a hostname, and enable networking so there is internet on every boot
     - Set a machine name and hosts file, enter these commands in sequence:
     `echo 'arch-vm' > /etc/hostname`
      `cat <<EOF >/etc/hosts`
      `127.0.0.1 localhost`
      `::1 localhost`
      `127.0.1.1 arch-vm.localdomain arch-vm`
      `EOF`
      - Enter `systemctl is-enabled NetworkManager` to verify. It output "enabled" so everything is good!
      - Enabling networkmanager now ensures GUI will have internet once it is installed later
  6. Initramifs
     - Enter `mkinitcpio -P` to rebuild.
     - This output some warnings, but I proceeded anyways. It ended up being okay.
 4. Password for Root User
     - Only used for administrative tasks, non-root users will be created later
     - Enter `passwd`
     - You will be asked for a password input and confirmation. The text you input will not appear. This freaked me out for a while until a quick google searched told me it was normal.
 5. Boot Loader
    - Install and configure a UEFI bootloader so the VM can boot the new Arch system.
    - I chose systemd-boot, its good and simple for VM's
    - Enter `bootctl --path=/boot install`to install bootloader to efi system partition. It will give security warnings but they are harmless for VM's.
    - Create loader config by entering the following commands:
    `cat >/boot/loader/loader.conf <<'EOF'`
    `default arch.conf`
     `timeout 3`
   `console-mode keep`
     `EOF`
    - Get the root partition's PARTUUID by entering `blkid -s PARTUUID -o value /dev/sda2` which output **4129a5f1-923d-4fe2-8b04-2de41f5c0652**
    - Enter the following commands:
    `cat >/boot/loader/entries/arch.conf <<'EOF'`
     `title Arch Linux`
     `linux /vmlinuz-linux`
     `initrd /initramfs-linux.img`
     `options root=4129a5f1-923d-4fe2-8b04-2de41f5c0652 rw`
     `EOF`
 6. Finish System Configuration
    - Exit chroot by entering `chroot`
    - Unmount by entering `umount -R /mnt`
    - Reboot into new arch system by entering `reboot`
    - Shut down VM
    - Eject disk from virtual drive in VirtualBox settings
    - Reboot from the new disk!

## Extra Project Steps
1. Boot into the New Disk
    - It will ask for a login
    - Enter username **root**
    - Enter the password created earlier
 2. Create Users (myself and Codi) and Give Sudo Permissions
     - Install sudo package by entering `pacman -S sudo`
     - Add brit as a user by entering `useradd -m -G wheel -s /bin/bash brit`
     - Enter `passwd brit`. The same password procedure from earlier will follow.
     - Repeat the last two commands for Codi.
 3. Allow Wheel Group Users to Use Sudo
     - Enter `EDITOR=nano visudo` to open sudoers file with safe editor.
     - Scroll down until **# %wheel ALL=(ALL:ALL) ALL** is found. It's in the bottom third of the file.
     - Uncomment by removing the **#**
     - Press `CTRL+O`, then `Enter`, then `CTRL+X`
     - Now Brit and Codi can use sudo for admin tasks.
     - We are now back in the shell prompt
 4. Install a New Shell
     - I chose Zsh, it is popular and customizable
     - Enter `pacman -S zsh`
     - Make it the default shell for both users brit and codi by entering `chsh -s /bin/zsh brit` and `chsh -s /bin/zsh codi`
 5. Install SSH
    - Allows to remotely log in to the VM
    - Enter the install command `pacman -S openssh`
    - Enable the SSH service by entering `systemctl enable sshd` and `systemctl start sshd`
    - Verify it is running by entering `systemctl status sshd`. It says it is enabled and running!
 6. Add Color
     - Enter `sed -i 's/^#Color/Color/' /etc/pacman.conf`
     - This uncomments the line that tells the pacman package manager to show colored output for installs and updates
 7. Install GUI
     - I am using LXQT for simplicity
     - Enter `pacman -S xorg sddm lxqt`
     - It will ask for specifications on installation. I kept pressing enter to accept the defaults. It will take a few moments to install
     - Enter `systemctl enable sddm` to set the login screen (Simple Desktop Display Manager) to start automatically on boot
     -  From there, I will log into LXQT graphically
2. Add Useful Aliases
     - Edit the file by entering `nano /home/brit/.zshrc`
     - I added the following aliases:
      `alias ll='ls -alF --color=auto'`
      `alias update='sudo pacman -Syu'`
      `alias cls='clear'`
      `alias grep='grep --color=auto'`
     `alias df='df -h'`
     - Once added, press `CTRL+O`, then `ENTER`, then `CTRL+X`
     - Switch to user brit by entering `su - brit`
     - Enter `nano ~/.zshrc` to create the config file
     
**NOTE:** I tried repeating the last two steps for root user, but it didn't work. I decided to just move on.

9. Reboot - There is now a working GUI!

## Conclusion
This project helped me understand how Linux bootloaders, partitions, and localization work at a low level. I learned to troubleshoot common installation errors (like timezone issues and verification errors) and gained confidence configuring a system manually rather than relying on automated installers.  
             
    

