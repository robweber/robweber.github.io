---
layout: post
title:  "Building Custom LibreELEC Images"
author: robweber
categories: automation coding
tags: kodi
---

We use [Kodi][kodi-link] for a lot of our media center needs in our house. We have several TVs, each with their own Kodi box, configured to pull information from a centralized database and shared network storage. To run Kodi I've used pre-made [LibreELEC images][libreelec-link]. LibreELEC is "just enough OS" system that is designed to run Kodi within a stripped down Linux OS designed just for that purpose. It also layers in some additional patches and features that make the whole experience easy to work with.

I've also been following work on Kodi's [RetroPlayer project][retroplayer-link]. This is an integration with Kodi and the excellent [RetroArch](https://www.retroarch.com/) system for game system emulation. Out of the box Kodi has many RetroPlayer features built in; but the RetroPlayer project is often well ahead of what Kodi has merged into it natively. For a long time I was able to hunt down LibreELEC builds that included these enhancements but I've been having a hard time with that lately. I've decided to tackle building my own custom LibreELEC builds that incorporate the latest RetroPlayer releases.

<!--more-->

{% include toc %}

## Building LibreELEC

Before customizing anything I wanted to make sure I could even build a base LibreELEC system. There are some good instructions online; but like most things it wasn't a completely smooth experience. First I forked the [LibreELEC.tv repository][libreelec-repo] and read through the [basic build instructions][build-instructions]. To test this I spun up an Ubuntu server, which is the preferred system based on the instructions. The very first step, installing the dependencies, threw a bunch of errors.

```
E: Unable to locate package makeinfo
E: Unable to locate package mkfontscale
E: Unable to locate package mkfontdir
E: Unable to locate package bdftopcf
E: Unable to locate package java
```

Some digging on their forums told me to just ignore these and the build system would find the deps later. I moved on to the next instruction but decided to fix that issue later. I cloned my fork of the project to the server but before building switched branches. The version of RetroPlayer I wanted to build is based on Kodi 19.4. The LibreELEC main branch is past that already so I checked out the 10.0 branch, which is based on Kodi 19.

```bash
git fetch
git branch -v -a
git checkout -b libreelec-10.0 remotes/origin/libreelec-10.0
```

I confirmed I had the right branch and then ran the download tool and project builder. I'm building a Generic version of the system but you can specify [other architecture types](https://wiki.libreelec.tv/development-1/build-commands/build-commands-le10) at this point.

```
PROJECT=Generic ARCH=x86_64 tools/download-tool
PROJECT=Generic ARCH=x86_64 make image
```

The `make image` command gave me a warning about the missing dependencies from earlier and asked to install them. I said yes to this and let the system build. The build docs will tell you, and it's true, that this will take a long time. I didn't clock it but plan on 5+ hours for sure depending on your system. If everything works properly you'll end up with a `target/` directory that has several files; including a tar and image file that you can use to install the system.

## Customizing The Build


### RetroPlayer Source

Once I knew I could build a base system the next step was building a RetroPlayer specific build. LibreELEC makes this pretty easy but you have to modify a few files to grab the modified Kodi build as part of the build chain. RetroPlayer releases to a [fork of the main Kodi repository](https://github.com/garbear/xbmc/releases). Looking at this page there are links to the source tarballs of this build under the Assets area. The link for the most recent one looked like this:

```
https://github.com/garbear/xbmc/archive/refs/tags/retroplayer-19.4-20220302.tar.gz
```

![RetroPlayer Build Page](/images/2022-03-29/retroplayer-source-link.png)

### Updating LibreELEC Build Files

Back in the LibreELEC repository folders I made a custom branch from the 10.0 branch as some files are going to need to be modified. I wanted to make sure I could backport any upstream changes in the future from the main LibreELEC project. To do this just run `git checkout -b retroplayer_custom`. This is now a branch of a branch. The file that needs to be modified to pull in the custom build is `packages/mediacenter/kodi/package.mk`. Below is the top part of the updated file with the original contents commented out.

* `PKG_VERSION` - becomes the RetroPlayer version from the link above.
* `PKG_SHA256` - can be commented out, or updated if you want to calculate the SHA256 hash.
* `PKG_URL` - changed to reflect the download location of the custom tar archive.

```
# SPDX-License-Identifier: GPL-2.0-or-later
# Copyright (C) 2009-2016 Stephan Raue (stephan@openelec.tv)
# Copyright (C) 2017-present Team LibreELEC (https://libreelec.tv)

PKG_NAME="kodi"
# PKG_VERSION="19.4-Matrix"
PKG_VERSION="retroplayer-19.4-20220302"
# PKG_SHA256="cc026f59fd6e37ae90f3449df50810f1cefa37da9444e1188302d910518710da"
PKG_LICENSE="GPL"
PKG_SITE="http://www.kodi.tv"
# PKG_URL="https://github.com/xbmc/xbmc/archive/${PKG_VERSION}.tar.gz"
PKG_URL="https://github.com/garbear/xbmc/archive/refs/tags/${PKG_VERSION}.tar.gz"
PKG_DEPENDS_TARGET="toolchain JsonSchemaBuilder:host TexturePacker:host Python3 zlib systemd lzo pcre swig:host libass curl fontconfig fribidi tinyxml libjpeg-turbo freetype libcdio taglib libxml2 libxslt rapidjson sqlite ffmpeg crossguid libdvdnav libhdhomerun libfmt lirc libfstrcmp flatbuffers:host flatbuffers libudfread spdlog"
PKG_LONGDESC="A free and open source cross-platform media player."
PKG_BUILD_FLAGS="+speed"

```

After saving this file I ran the `make image` command from above again to build the new image. This failed almost instantly as the LibreELEC custom patches to Kodi could not be applied properly. Apparently there are some custom changes made to the RetroPlayer branch that modified the source just enough that one LibreELEC patch couldn't be applied. The problem file was `packages/medicenter/kodi/patches/kodi-100.31.le-addons-no-startupenable.patch`. Comparing to [the source](https://github.com/garbear/xbmc/blob/retroplayer-19.4/xbmc/platform/linux/PlatformLinux.h) the line numbers and part of the function definition were different than what the patch expected. I figured I could either a) modify the patch to work or b) delete the patch and live without it. To my eyes the RetroPlayer branch changes basically did the same thing as the patch so I think it would have been fine either way. I decided to modify the patch, mostly to keep consistency with LibreELEC mainline. This involved editing the file above to change the line number and matching function definition.

Now running the make command seemed to work. This build ran a lot faster as only the new Kodi build needs to be compiled instead of the entire project.The result was a set of files in the `target/` directory that I could use to deploy the RetroPlayer version of Kodi.

## Adding Tweaks

I honestly could have stopped here as I had working build but I didn't. I decided to add a few helpful things so that I could continue to easily build this project as Kodi versions changed.

### Build Dependencies

As noted above the build dependencies had an error trying to install prior to running the make command. These were resolved during the build but I had to hit `y` to confirm first. In the future I wanted to be able to script this process and not touch it so I made some changes to the `scripts/checkdeps` file. I [removed the prompt](https://github.com/robweber/LibreELEC.tv/blob/retroplayer_custom/scripts/checkdeps#L38) so this could be run manually or as part of a larger script that I planned to write to wrap this all together. When run via the make command it will simply skip the prompt and attempt to install them - which does require __sudo__ if you're not a root user.

### Advanced Build Params

LibreELEC allows for some [advanced build parameters](https://wiki.libreelec.tv/development-1/build-advanced) to further customize the build. I added some of these so that I could better keep track of what version of Kodi and RetroPlayer I had for future reference.

```
PROJECT=Generic ARCH=x86_64 BUILDER_NAME=robweber BUILDER_VERSION=20220302 make image
```

### Build Script

To wrap everything together I created a [build script](https://github.com/robweber/LibreELEC.tv/blob/retroplayer_custom/create_build.sh) that I could execute after cloning the repo and kick off the build process without having to manually do a bunch of the steps above. As a bonus I can set the package name here that will set the `package.mk` variable to update the Kodi package with whatever the most current RetroPlayer build is.

```bash
#!/bin/bash
# Update the RetroPlayer build name and build the LibreELEC image
# Rob Weber

RETROPLAYER_VERSION="retroplayer-19.4-20220302"
KODI_PACKAGE_FILE="packages/mediacenter/kodi/package.mk"

# install min deps needed to kick start build system
#sudo apt-get update
#sudo apt-get install -y gcc make git unzip wget xz-utils bc gperf zip unzip g++ xsltproc

# Modify the Kodi Package file
sed -e "s/@RETROPLAYER_VERSION@/${RETROPLAYER_VERSION}/" -i ${KODI_PACKAGE_FILE}

# create the image
PROJECT=Generic ARCH=x86_64 BUILDER_NAME=robweber BUILDER_VERSION=${RETROPLAYER_VERSION} make image

# Revert the kodi file
git checkout ${KODI_PACKAGE_FILE}
```

## Conclusion

The end result is pretty much what I wanted; a build of LibreELEC based on the RetroPlayer branch for all of my Kodi boxes. I did an in place upgrade from my current system and things worked pretty smooth.

To date I haven't tried building any of the other architectures yet. I do have an Raspberry Pi that I used for a "travel system" when I travel for work but haven't updated it yet. Hopefully as the main LibreELEC repo is updated I can pull in changes to my custom branch and rebuild without too much effort.

## Links

* [LibreELEC Main Repo][libreelec-repo]
* [RetroPlayer Kodi Repo](retroplayer-link)
* [LibreELEC Build Instructions][build-instructions]
* [robweber Custom RetroPlayer Repo](https://github.com/robweber/LibreELEC.tv/tree/retroplayer_custom)


[kodi-link]: https://kodi.tv
[libreelec-link]: https://libreelec.tv
[libreelec-repo]: https://github.com/LibreELEC/LibreELEC.tv.git
[retroplayer-link]: https://github.com/garbear/xbmc
[build-instructions]: https://wiki.libreelec.tv/development-1/build-basics
[package-layout]: https://github.com/LibreELEC/LibreELEC.tv/blob/master/packages/readme.md
