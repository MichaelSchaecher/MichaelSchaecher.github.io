---
layout: post
title: How to Install WSL2 for Windows 10/11 and Archlinux
author: main_author
date: 2023-05-11
image: /assets/images/How-to-Install-Archlinux-in-WSL2/archlinux.png
categories: [WSL2, Linux]
tags: [Arch]
comments: true
toc: true
---

Windows Subsystem for Linux (WSL) allows you to run Linux distributions on Windows, giving you access to a powerful and flexible operating system within the Windows environment. Arch Linux (known for its lightweight and highly customizable nature) is a popular choice among Linux enthusiasts and developers.

In this tutorial, your'll be guide you through the process of installing WSL2 on Windows 10/11 and then show you how to install Arch Linux within it. We'll cover everything from enabling the necessary Windows features, updating the Linux kernel, setting up Arch Linux, and configuring the system. By following this guide, you'll be able to use Arch Linux on your Windows machine via WSL2, opening up a world of possibilities for development and customization.

This guide is for anyone that is like me and have no choice but to use Windows has their daily driver. So, let's get started!

## Setting WSL2 and Home User Environment

One of the requirements to installing the ArchLinux CLI is having the environment setup according and not sticking with the install defaults for WSL.

Setting the Windows environment for WSL involves enabling the Windows Subsystem for Linux (WSL) feature, installing a Linux distribution of your choice, and setting up a terminal application to interact with the Linux environment. This can be done easily through the Windows settings, Microsoft Store, and package managers such as Chocolatey or Winget. Once set up, you can use the Linux environment on your Windows machine to run Linux commands and applications, giving you access to a powerful and flexible operating system within the Windows environment.

### Some Requirements To Keep In Mind

If you plan on installing WSL for each user account they needs to have administrator privileges on the Windows machine. The administrator can enable the Windows Subsystem for Linux (WSL) feature through the Windows settings, and then each user account can install a Linux distribution of their choice from the Microsoft Store or by manually importing a distribution image. Each user account can also set up their own terminal application to interact with their WSL environment.

Each user account's WSL environment can be managed independently, including updating, installing, or removing Linux distributions, and configuring WSL settings. Each user account's WSL environment will have its own file system and home directory, separate from other user accounts on the same machine.

### Picking a Location


The default install location is to place the distro where [Windows Store](https://apps.microsoft.com/store/apps) gets installed. This is because the distro's are infact store apps. The downside to installing this way is that yet another terminal installed. The way around this is to import the distro into _WSL_.

There is a downside to this, the tarball most be of the root of the distro's bootstrap which **Arch** is not. You can extract the tarball then recompress inside the root directory, however this cannot be down outside of a **Linux** based filesystem. Meaning the _NTFS_ filesystem well cause errors. This why an already recompressed tarball has been made available [here](https://github.com/MichaelSchaecher/MichaelSchaecher.github.io/releases/download/Archlinux-x86_64/archlinux-x86_64.tar.gz). Making it easier to install **Arch** if you don't have a distro available.

Thankfully there is no default install location for imported distributions, meaning that the choice is up to you. The best place is to setup a directory in you home fold where everything _WSL_ can live; including any kernels, shared virtual drives and backups.

From the _shell_ profile of _Windows Terminal_ move to the home directory with `cd $HOME`.

```powershell
Invoke-WebRequest -Uri "https://github.com/MichaelSchaecher/MichaelSchaecher.github.io/releases/download/Archlinux-x86_64/Archlinux-Bootstrap-x86_64.tar.gz" -OutFile "$HOME\Downloads\archlinux-x86_64.tar.gz"
```

Please note that you can pick where the file is Downloads to and what to name the file. This is done before moving on to the next set as to not get confused.

Create the root install folder and general directory structure for all _WSL_ imported distributions with `md "wsl"`.

```powershell
'kernel', 'distributions\arch', 'distributions\ubuntu', '.backup' | % {New-Item -Name "wsl\$_" -ItemType 'Directory'}
```

Just note that _WSL_ well only allow each user with permissions to have their own running account. This means that multi-user login pre installed distribution well not work.

Lately import to **Arch** tarball into _WSL_.

```powershell
wsl --import Arch "$HOME\WSL\Arch" "$HOME\Downloads\arch_linux-x86_64.tar.gz"
```

## Setting Up Arch

Once that **Arch** is installed run `wls -d Arch`, you well be greeted with the root user prompt and current **Windows** working directory. Some of what is going to be done maynot be possible in this location, to be on the save side _cd_ to the home directory with `cd ~`, this well be the `/root` folder.

In some cases you well see network problems until the ping command is set. Though this should be rare, it is nice to be able to troubleshut network problems with a simple ping.

```bash
setcap 'cap_net_raw+p' /bin/ping
```

From here the fun can begin, starting with getting _pacman_ to function properely and enable multilib for development.

```bash
sed -i 's:#Server:Server:g' /etc/pacman.d/mirrorlist
```

```bash
linenumber=$(grep -nr "\\#\\[multilib\\]" /etc/pacman.conf | gawk '{print $1}' FS=":")
sed -i "${linenumber}s:.*:[multilib]:" /etc/pacman.conf
linenumber=$((linenumber+1))
sed -i "${linenumber}s:.*:Include = /etc/pacman.d/mirrorlist:" /etc/pacman.conf
```

Populate the keyring.

```bash
pacman-key --init && pacman-key --populate
```

If you try to update without populating to keyring first you well get permission errors.

Once the keyrings are populated **Arch** can be updated. This needs to be done before setting you user account and installing additional packages.

```bash
pacman -Syu --noconfirm
```

Now install the additional packages.

```bash
pacman -S --noconfirm base-devel git vim nano wget reflector sudo which go openssh man-db shell-completion fontconfig
```

Configure _pacman_ to use only the most current and best connection mirror.

```bash
reflector --country "United States" --age 12 --protocol https --sort rate --save /etc/pacman.d/mirrorlist
```

## User Account

Setting up a user account is similar to installing **Arch** on bare-metal. The first thing to do is editing the `/etc/sudoers` file to enable running some basic commands that required root permissions.

```bash
export EDITOR=nano ; visudo
```

or

```shell
export EDITOR=vim ; visudo
```

Uncomment the line containing `%wheel ALL=(ALL) ALL`.

![sudoers](/assets/images/How-to-Install-Archlinux-in-WSL2/uncomment-suders-file.png)

If you wish to run basic commands that require root permissions without having to enter user password then uncomment the next `%wheel` group.

Now that _sudo_ is setup it is time to set the default user account that will be singed into when logging into **Arch**.

```bash
useradd -m -G wheel -s /bin/bash -d /home/username username ; passwd username
```

Replace 'username' with your accult name, just keep in mind that no spaces in username. The password can be anything that you wish including the password that you use to log into Windows. Just keep in mind that weather the password is shared with others.

Unlike with distributions installed from the store, imported distroes will not have a default log in account set. This need to be set with a `/etc/wsl.conf` file in order for _WSL_ to know that a default user has been set.

```ini
[automount]
enabled = true
options = "metadata,uid=1000,gid=1000,umask=22,fmask=11,case=off"
mountFsTab = true
crossDistro = true

[network]
generateHosts = false
generateResolvConf = true

[interop]
enabled = true
appendWindowsPath = true

[user]
default = username

[boot]
systemd = true
```

Just remember to use the account name that your setup earlier.

With the user account set and the `/etc/wsl.conf` edited it is time to log into the accout to finish up the install. to do this **Arch** needs to be shutdown with `wsl --shutdown Arch` after exiting.

Close the terminal.

Reopen _Windows Terminal_ next to the '+' symbol you with '⌄': left click and go to settings and locate **Arch** under "Profiles" and select. In "Starting Directory" change to `\\wsl$\Arch\home\username`. Just replace username with accult name and click save.

## Enable AUR with YAY

The best part and sometimes problem with **Arch** is the AUR (Arch User Repository) and having access to that great resource can be benefit. However, in order to employee the AUR can be quite a task without an application like _YAY_. Setting up _YAY_ is easy and once done both the default **Arch** repository and AUR came checked and update simply.

From the user home directory run.

```bash
git clone https://github.com/Jguer/yay.git
cd yay
makepkg -si
```

Remove the yay direcotry and enjoy **Arch** distro running on _WSL_.
