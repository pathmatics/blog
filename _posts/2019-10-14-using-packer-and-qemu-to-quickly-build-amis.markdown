---
layout: post
title:  "Using Packer and QEMU to Quickly Build AMIs/VM Images"
date:   2019-10-14 16:00:00 -0600
categories: packer qemu ami aws
---

[Packer](https://www.packer.io/) is a super useful tool to help you create and modify AMIs and VM snapshots. All tools and scripts to generate your images can be committed to source control, and even wired up to your CI/CD tooling to allow you to fully trust your images (a post about this is coming soon).

At Pathmatics, we leverage AWS AMI's to launch fleets of EC2 instances, and in the past, we've tried to keep a manual record of how to build a base iamge for all our software. Frankly, this was error prone and didn't scale. We needed a way to have an enforcable, consistent way of building an image, and Packer was the answer. There are many great tutorials out there on how to build an AMI with Packer (starting with [Packer's](https://www.packer.io) own documentation), but one of our frustrations with using the AMI builder was the slow iteration time. It was pretty irritating to discover a syntax error in a Bash or Powershell script af the tail end of a long AMI build process.

Our solution to this was to leverage a different type of builder that Packer also supports: [QEMU](https://www.qemu.org/). QEMU is an emulator that can run on top of a bunch of different virtualization engines (in our case Hyper-V). Using the QEMU builder, we were able to snapshot the VM at any point in the provision script on our local dev machines and take the pain out of making a mistake in our provisioner scripts.

We found it useful to wrap our calls to the Packer binary in a script so we could parameterize what was the base snapshot. This allowed us to get our image to a good point, snapshot it, then add additional code. The only catch was that your provision script needs to be idempotent. But this saved us a ton of time where our image would be installing apt packages over and over again.