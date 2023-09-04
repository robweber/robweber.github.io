---
layout: post
title:  "Home Lab Backups"
author: robweber
categories: automation smarthome random
tags: atom chocolatey home-assistant kodi
---

For as long as I can remember I've always been really paranoid about losing data. When Windows XP was brand-spanking new it was having two hard drives in my PC; in college I cobbled together a RAID system from an old Intel 3 tower and now it's pushing files to dedicated NAS hardware or in the cloud. I'm also sort of lazy; at least about re-doing work I've already done before. While Git repositories work great for my coding projects there are a lot of other things that need redundancy as well. With that in mind I'll highlight a few things I do to try make sure I can recover systems when I need to.

__There are hundreds of different ways you could set this kind of thing up but this is what works for me.__ If you just want the list of software jump right to the [links](#links).

<!--more-->

{% include toc %}

## Desktop PC

### Chocolatey

For my desktop PC I'm a big fan of [Chocolatey][chocolatey]. Chocolatey is a PowerShell based package manager for Windows based on NuGet. It's simple to use and has a strong [package repository](https://community.chocolatey.org/packages). Pretty much all of the tools I use can be found there and easily installed. A few helpful commands are:

```powershell
# install a package from the repo
choco install packagename

# list outdated packages
choco outdated

# update packages
choco update all -y

# list all locally installed packages
choco list -lo

```

I use the command `choco list -lo` to pull in a list of all installed software and keep it in a GitHub Gist as reference.

### Dropbox

Everyone has their favorite flavor of cloud storage. I usde [Dropbox][dropbox] as my preferred file storage backup. Mostly this is due to the fact that it has a [Linux daemon](https://www.dropbox.com/install-linux) and a good [Python library](https://github.com/dropbox/dropbox-sdk-python).

### Veeam
Additionally I do a full metal backup of the entire machine using Veeam's free [Windows Agent][veeam-agent], which can also be installed via [Chocolatey](https://community.chocolatey.org/packages/veeam-agent). This will do an automatic full metal backup of my desktop to my NAS each day.

### Atom IDE
For most development these days I use [Atom][atom]. I've tried lots of other IDEs over the years, and still prefer Eclipse for strictly Java development, but Atom is my go-to at the moment. Since I do development on a number of computers I also use the [sync-settings][atom-sync] package to sync my Atom preferences specifically. This works great in quickly getting a Dev environment setup on a different computer.

One other package I'd like to call out is the [remote-editor](https://atom.io/packages/remote-editor) package. I often have projects that _run_ on Linux systems but like to develop on my Windows desktop so that I can use the IDE, do web searches, respond to emails, pretend to be productive. remote-editor allows me to use an SSH connection to get in to my Linux environment but do the actual file editing locally. This is especially useful for Raspberry Pi based projects that run headless.

## Home Lab

For hardware in my home lab I use a Synology NAS alongside a tower PC I've converted to a VMware hypervisor.

### Git

For the VMs I keep any install scripts or instructions stored in GitHub in a private repo. This makes re-creating a device super easy and _near_ hands-off should I need to start from scratch. Most systems are running processes with static config files so this works well. The layout of the repo is:

```
configs/
  hostname/
    file1.conf
    file2.conf
  hostname2/
    file1.conf
mysql/
  hostname/
    db.sql
scripts/
  base.sh
  hostname.sh
README.md
setup.sh
```

The `setup.sh` script in the root of the directory sets up the basics for the entire process. I download this with a `wget` command using a direct URL to the file. After that I run `chmod +x setup.sh` and run the file. This downloads some OS packages, like git, and clones the full repo. From here I can run the `base.sh` script to install any base system components and then the `hostname.sh` file for whatever host I'm working with. The script references config files from the other directories as programs are installed.

### ghettoVCB

The install scripts work when doing things from scratch; but storing actual system backups on a NAS is much faster for recovery. For the VMs I use [ghettoVCB][ghetto-vcb] to do full backups of each system. ghettoVCB works in much the same fashion as expensive hypervisor backups; but without a fancy interface. It's simple to use and open source so I don't have to worry about licensing.

The project does have a learning curve so I've found it's best to fork it and store your personal configuration that way. For my personal setup I run the following command to the script:

```bash
# add -d dryrun to do a test run
./ghettoVCB.sh -g globalConf.conf -f backup_list.txt -c backup_configs
```

The `-g` option points to a global config file. This is a copy of their default file but with the path to my backup storage and other options in it. `-f` is a text file with the name of each VM I want to back up. Finally the `-c` option is a folder that contains custom per-VM instructions. Normally this isn't needed but I have some machines where I only want certain disks uploaded.

### Home Assistant

To control my smart home setup I utilize [Home Assistant](https://www.home-assistant.io/). Backing up this system can be done in a number of ways but [this blueprint](https://community.home-assistant.io/t/create-automated-backups-every-day/254039) can be imported to help automate the task each day. Once it's done there are numerous addons available to copy the file off your main system. A few examples of these are below:

* [Backup to NAS](https://community.home-assistant.io/t/samba-backup-create-and-store-snapshots-on-a-samba-share/199471)
* [Backup to Google Drive](https://community.home-assistant.io/t/hass-io-add-on-hass-io-google-drive-backup/107928)
* [Backup To Dropbox](https://community.home-assistant.io/t/hass-io-add-on-upload-hassio-snapshots-to-dropbox/43622)

## Media Center

I utilize the [Kodi](https://www.kodi.tv) media center for watching Live TV and local video content. To keep the configurations on these devices backed up I use my own Kodi addon, [Backup][kodi-backup]. This addon can be found in the Kodi Addon Repository and installed directly from within Kodi.

## Update 7/7/2023

Since this post was written things have changed (big suprise) that have altered my setup a bit.

### Atom Sunset

I know everyone is all on board with VS Code, I'm just not there for whatever reason. When Atom was sunset I was honestly pretty bummed. I had my setup exactly how I wanted it and didn't want to learn a new IDE. Enter [Pulsar][pulsar]. Pulsar is an attempt to keep the Atom project alive through the Open Source community. Not sure if it will stand the test of time but it's active for now and releasing versions fairly regularly.

It's almost a 1:1 drop in replacement for Atom. You can even use most of the same plugins since it's scraped and re-built the original Atom plugin infrastructure. I've found a few that didn't work but the core of what I needed was available. Until somone can convince me to get on the VS Code bandwagon this will satisfy my childish impulse not to change.

### Terminal Software

I've also started using [Tabby][tabby] as my terminal program of choice. I used to use Putty, or one of it's offshoots, but I've come to really like how Tabby can keep all my various terminals (including Powershell when on Windows) into one program. Plus it has some great out of the box features and plugins.

Like a lot of programs Tabby has [a plugin](https://www.npmjs.com/package/terminus-sync-config) that will offload your config to a Gist. Just like with Atom/Pulsar this is super convienent for moving between different working environments.

### Ansible

Probably the biggest change is finally embracing [Ansible][ansible] in my home lab. Professionally I've used other tools, usually Puppet, but never jumped in to any kind of config management on my home systems. I'm still in the early steps of learning but am slowly migrating old config files and scripts into the Ansible environment. This should be more future-proof than scripts and I can still keep everything in my Git repo.

## Links

* [Pulsar][pulsar] ~~[Atom][atom]~~ - a full featured, hackable, text editor
  * [Pulsar - sync-settings][pulsar-sync] ~~[Atom - sync-settings][atom-sync]~~ - Pulsar ~~Atom~~ package to sync settings to GitHub Gist
* [Chocolatey][chocolatey] - Windows package installer
* [Dropbox][dropbox] - cloud storage
* [ghettoVCB][ghetto-vcb] - VMware hypervisor backup
* [Kodi- Backup][kodi-backup] - Backup addon for the Kodi media center
* [Veeam Agent][veeam-agent] - free full metal backup solution for desktops and servers
* [Tabby][tabby] - cross platform terminal app for local, serial, SSH and Telnet
* [Ansible][ansible] - tool that enables infrastructure automation

[atom]: https://atom.io/
[atom-sync]: https://atom.io/packages/sync-settings
[chocolatey]: https://chocolatey.org/
[dropbox]: https://www.dropbox.com
[ghetto-vcb]: https://github.com/lamw/ghettoVCB
[kodi-backup]: https://github.com/robweber/xbmcbackup
[veeam-agent]: https://www.veeam.com/windows-endpoint-server-backup-free.html
[pulsar]: https://pulsar-edit.dev/
[tabby]: https://tabby.sh/
[ansible]: https://www.ansible.com/
[pulsar-sync]: https://web.pulsar-edit.dev/packages/sync-settings
