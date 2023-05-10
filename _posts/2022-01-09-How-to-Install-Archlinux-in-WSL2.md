---
layout: post
title: How to Install Archlinux in WSL2
subtitle: With AUR Support
last-updated: 2022-09-30
thumbnail-img: /assets/images/posts/How-to-Install-Archlinux-in-WSL2/archlinux.png
cover-img: /assets/images/posts/How-to-Install-Archlinux-in-WSL2/archlinux.png
tags: [Arch, Windows, WSL2, Linux]
text-align: justify
comments: true
published: true
---

This guide is for anyone that is like me, an **Arch** daily driver user but do to circumstances is suck using **Windows**. Originally this was center on getting _SystemD_ to work inside _WLS2_ while running **Archlinux**, however do to changes made to some packages relating daemons and services _SystemD_ is left degrade. There is news that _WSL2_ by default well support _SystemD_ with some version of **Windows** 10/11. So until then, consider _SystemD_ unsupported.

Most commands can be done in the _shell_ cli alone, but _Windows Terminal_ is a must have for and **Windows** PC as even _WSL_ distro's  can have there own profile. If you have not done so already install the terminal, go [here](https://github.com/microsoft/terminal) to learn how using the _Windows_ packages manager `winget`.

## Getting Environment Read

The default install location is to place the distro where [Windows Store](https://apps.microsoft.com/store/apps) gets installed. This is because the distro's are infact store apps. The downside to installing this way is that yet another terminal installed. The way around this is to import the distro into _WSL_.

There is a downside to this, the tarball most be of the root of the distro's bootstrap which **Arch** is not. You can extract the tarball then recompress inside the root directory, however this cannot be down outside of a **Linux** based filesystem. Meaning the _NTFS_ filesystem well cause errors. This why an already recompressed tarball has been made available [here](https://github.com/MichaelSchaecher/MichaelSchaecher.github.io/releases/download/Archlinux-x86_64/archlinux-x86_64.tar.gz). Making it easier to install **Arch** if you don't have a distro available.

Thankfully there is no default install location for imported distributions, meaning that the choice is up to you. The best place is to setup a directory in you home fold where everything _WSL_ can live; including any kernels, shared virtual drives and backups.

From the _shell_ profile of _Windows Terminal_ move to the home directory with `cd $HOME`.

```shell
Invoke-WebRequest -Uri "https://github.com/MichaelSchaecher/MichaelSchaecher.github.io/releases/download/Archlinux-x86_64/Archlinux-Bootstrap-x86_64.tar.gz" -OutFile "$HOME\Downloads\archlinux-x86_64.tar.gz"
```

Please note that you can pick where the file is Downloads to and what to name the file. This is done before moving on to the next set as to not get confused.

Create the root install folder and general directory structure for all _WSL_ imported distributions with `md "wsl"`.

```shell
'kernel', 'distributions\arch', 'distributions\ubuntu', '.backup' | % {New-Item -Name "wsl\$_" -ItemType 'Directory'}
```

If you gone the route of setting up a .backup it maybe a good idea to make it hidden.

```shell
$f=get-item $HOME\wsl\.backup -Force
$f.attributes="Hidden"
```

Just note that _WSL_ well only allow each user with permissions to have their own running account. This means that multi-user login pre installed distribution well not work.

Lately import to **Arch** tarball into _WSL_.

```shell
wsl --import Arch "$HOME\WSL\Arch" "$HOME\Downloads\arch_linux-x86_64.tar.gz"
```

## Setting Up Arch

Once that **Arch** is installed run `wls -d Arch`, you well be greeted with the root user prompt and current **Windows** working directory. Some of what is going to be done maynot be possible in this location, to be on the save side _cd_ to the home directory with `cd ~`, this well be the `/root` folder.

In some cases you well see network problems until the ping command is set. Though this should be rare, it is nice to be able to troubleshut network problems with a simple ping.

```shell
setcap 'cap_net_raw+p' /bin/ping
```

From here the fun can begin, starting with getting _pacman_ to function properely and enable multilib for development.

```shell
sed -i 's:#Server:Server:g' /etc/pacman.d/mirrorlist
```

```shell
linenumber=$(grep -nr "\\#\\[multilib\\]" /etc/pacman.conf | gawk '{print $1}' FS=":")
sed -i "${linenumber}s:.*:[multilib]:" /etc/pacman.conf
linenumber=$((linenumber+1))
sed -i "${linenumber}s:.*:Include = /etc/pacman.d/mirrorlist:" /etc/pacman.conf
```

Populate the keyring.

```shell
pacman-key --init && pacman-key --populate
```

If you try to update without populating to keyring first you well get permission errors.

Once the keyrings are populated **Arch** can be updated. This needs to be done before setting you user account and installing additional packages.

```shell
pacman -Syu --noconfirm
```

Now install the additional packages.

```shell
pacman -S --noconfirm base-devel git vim nano wget reflector sudo which go openssh man-db shell-completion fontconfig
```

Configure _pacman_ to use only the most current and best connection mirror.

```shell
reflector --country "United States" --age 12 --protocol https --sort rate --save /etc/pacman.d/mirrorlist
```

## User Account

Setting up a user account is similar to installing **Arch** on bare-metal. The first thing to do is editing the `/etc/sudoers` file to enable running some basic commands that required root permissions.

```shell
export EDITOR=nano ; visudo
```

or

```shell
export EDITOR=vim ; visudo
```

Uncomment the line containing `%wheel ALL=(ALL) ALL`.

![sudoers](/assets/images/posts/How-to-Install-Archlinux-in-WSL2/uncomment-suders-file.png)

If you wish to run basic commands that require root permissions without having to enter user password then uncomment the next `%wheel` group.

Now that _sudo_ is setup it is time to set the default user account that will be singed into when logging into **Arch**.

```shell
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
```

Just remember to use the account name that your setup earlier.

With the user account set and the `/etc/wsl.conf` edited it is time to log into the accout to finish up the install. to do this **Arch** needs to be shutdown with `wsl --shutdown Arch` after exiting.

Close the terminal.

Reopen _Windows Terminal_ next to the '+' symbol you with '⌄': left click and go to settings and locate **Arch** under "Profiles" and select. In "Starting Directory" change to `\\wsl$\Arch\home\username`. Just replace username with accult name and click save.

## Enable AUR with YAY

The best part and sometimes problem with **Arch** is the AUR (Arch User Repository) and having access to that great resource can be benefit. However, in order to employee the AUR can be quite a task without an application like _YAY_. Setting up _YAY_ is easy and once done both the default **Arch** repository and AUR came checked and update simply.

From the user home directory run.

```shell
git clone https://github.com/Jguer/yay.git
cd yay
makepkg -si
```

Remove the yay direcotry and enjoy **Arch** distro running on _WSL_.
