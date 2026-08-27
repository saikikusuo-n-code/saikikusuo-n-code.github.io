---
title: "[SW.Modification] Windows - USB drive cannot format or create partition after using with Rufus and Linux ISO"
date: 2026-08-27 16:41:00
categories:
- Debugging
- Tutorial
---

# Issue?

You try to format partition for situation like reinstallation media where the partition must be FAT32 (<=32GB). The partition can be resized but cannot format with error like "The system cannot find the file specified"; both Disk Managment and Diskpart.

# Why it happens?

The partition table of the USB is altered by Rufus. If you have written linux ISO with Rufus in raw mode the partition can be accessed but modifying partition format will fail.

# How to solve?

The first 384 sectors need to be zeroed, use HXD to clean the partition table if Windows. If you use Linux:

``
dd if=/dev/zero of=/dev/XXX count=1 bs=4096
XXX = USB device
``

# Source

https://superuser.com/a/1139310