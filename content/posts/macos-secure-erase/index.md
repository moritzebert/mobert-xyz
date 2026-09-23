+++
title = 'Secure Erase on macOS 27 Golden Gate'
date = 2026-09-23T23:36:30+02:00
tags = ["storage"]
draft = false
+++

I'm giving my old Raspberry Pi 4 to someone, so the SD card needs a proper wipe first. I put the card into my MacBook, opened Disk Utility , selected the card, and clicked **Erase**. 

There was no **Security Options** button, the slider that used to let you choose how many times to overwrite the disk. [Jeff Geerling hit the same thing](https://www.jeffgeerling.com/blog/2026/securely-erase-hard-drive-macos-tahoe/) and his post pointed me to the fix. It seems to have disappeared around macOS 15 Sequoia.

The capability still exists in Terminal. Maybe they removed it because multi-pass erase makes more sense on a spinning hard drive. But who knows why they chose to remove it from the GUI.

## How to do it

First, find your card:

```bash
$ diskutil list
```

Check the size to be sure you have the right disk, and note the identifier (something like `/dev/disk4`).

First let's check the options for secure erase.

```bash
$ diskutil secureErase
Usage:  diskutil secureErase [freespace] level
        MountPoint|DiskIdentifier|DeviceNode
"Securely" (BUT SEE "man diskutil" FOR MODERN LIMITATIONS) erases either a
whole disk or a volume's freespace. Level should be one of the following:
        0 - Single-pass erase resulting in a zero fill.
        1 - Single-pass erase resulting in a random-number fill.
        2 - Seven-pass "secure" erase.
        3 - Gutmann algorithm 35-pass "secure" erase.
        4 - Three-pass "secure" erase.
Ownership of the affected disk is required.
Note: Level 2, 3, or 4 secure erases can take an extremely long time.
```

I will choose a single-pass with a random number - option 1. Replace the identifier with your own and **double-check it**, because this will destroy whatever disk you point it at. The number sets the method.

```bash
$ diskutil secureErase 1 /dev/disk4
```

32GB SD card took about 5 minutes. Multi-pass options will obviously take longer.

## One last tought

Because SD cards are flash storage, no overwrite can guarantee every bit is unrecoverable. That's fine as long as there's no truly sensitive data on the card, which was the case for mine. If yours did hold something sensitive, I would probably just physically destroy it. SD cards are cheap.