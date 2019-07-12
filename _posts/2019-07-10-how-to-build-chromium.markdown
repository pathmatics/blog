---
layout: post
title:  "How to Build Chromium Package in Ubuntu"
date:   2019-07-10 16:00:00 -0600
categories: chromium debian ubuntu linux
---
At Pathmatics, we leverage the open source Chromium browser in a lot of our tools. We fix on a particular version and then need to modify the source code in some way. This requires building custom Chromium packages for Ubuntu. Here I'll document how you accomplish that:

*This example was exectued on an AWS EC2 Ubuntu 18.04 instance sized i3.4xl*

First, switch to root:
```bash
sudo su
```

Next, install the required build packages:
```bash
apt update
apt upgrade
apt install git \
    devscripts \
    mdadm \
    bzr \
    bzr-builddeb \
    ubuntu-dev-tools \
    ninja-build \
    dh-buildinfo \
    pkg-config bison \
    clang \
    gperf \
    libpulse-dev \
    libcups2-dev \
    libasound2-dev \
    libnss3-dev \
    mesa-common-dev \
    libpci-dev \
    libxtst-dev \
    libxss-dev \
    libgtk-3-dev \
    libgnome-keyring-dev \
    libcap-dev \
    libgcrypt-dev \
    libkrb5-dev \
    libpam0g-dev \
    uuid-dev \
    chrpath \
    yasm
```
*mdadm isn't strictly required but we stripe the instance drives for increased I/O performance on build*

This next step is optional, but we stripe the EC2 SSD instances for faster I/O performance, thus faster builds
```bash
yes | mdadm --create /dev/md127 --level 0 --name=0 --raid-devices 2 /dev/nvme0n1 /dev/nvme1n1
mkfs -t ext4 /dev/md127
mount /dev/md127 /mnt
cd /mnt
```

From here on out, we'll assume you are going to build from the `/mnt` folder.

Now, you need to determine what version of the Chromium packages you want to build. We recommend you go with the latest stable, unless you have a good reason not too. Each successive version has security fixes that could leave you vulnerable if you run an older version. Navigate to the repo corresponding to your [Ubuntu distro for Chromium](https://bazaar.launchpad.net/~chromium-team/chromium-browser/bionic-stable/changes) and select the tag you want to build from. In our example, we'll build `69.0.3497.81-0ubuntu0.18.04.1`.

Pull down the source code like so:
```bash
pull-lp-source chromium-browser 69.0.3497.81-0ubuntu0.18.04.1
cd 69.0.3497.81-0ubuntu0.18.04.1
```

Now you have the source code, modify it as you see fit.

To build and package:
```bash
dpkg-buildpackage -rfakeroot -uc -b
```

Assuming all went well, your packages should be located in /mnt



