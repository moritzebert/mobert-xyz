+++
title = 'Automating Linux updates with UnattendedUpgrades on debian'
date = 2025-08-17T00:45:18+02:00
tags = ["linux","automation","security"]
draft = false
+++

Usually, I ssh into my debian server to run updates every now and then — maybe once every couple of weeks.  
Recently, I asked a colleague (who manages our Linux systems) about his preferred way to keep servers up to date. He recommended **[UnattendedUpgrades](https://wiki.debian.org/UnattendedUpgrades)**, a built-in feature of debian-based distributions that automatically installs important security patches without user intervention.

So I gave it a try.

## Installation

The package is called 'unattended-upgrades'. Installing it is straightforward.

```bash
sudo apt update && sudo apt install unattended-upgrades
```

During installation, it might ask if you want to enable automatic updates. If you skipped it or aren’t sure, you can reconfigure it by running:

```bash
sudo dpkg-reconfigure unattended-upgrades
```

## Configuration

The default config file is located at `/etc/apt/apt.conf.d/50unattended-upgrades`.
By default, it only applies security updates, but you can configure much more.

In my case, I wanted to enable:

+ Feature updates
+ Automatic removal of unused packages
+ Automatic reboots
+ Mail notifications on errors or package changes

Any custom settings should be placed in `/etc/apt/apt.conf.d/52unattended-upgrades-local`.

```bash
sudo nano /etc/apt/apt.conf.d/52unattended-upgrades-local
```

I aim to minimize maintenance — not because I’m lazy, but because I’d rather spend time on better things.
For this reason, I also enabled feature updates. While this isn’t always recommended for production servers, the risk of breakage is close to zero. The only issues I’ve ever run into were with Docker container updates that broke apps due to upstream changes.

### Enable feature updates

Add the following lines to your local config file:

```bash
Unattended-Upgrade::Origins-Pattern {
        "origin=Debian,codename=${distro_codename}-updates";
};
```

### Remove unused packages automatically

Instead of manually running `apt autoremove && apt clean`, let unattended-upgrades handle it.

```bash
Unattended-Upgrade::Remove-Unused-Dependencies "true";
```

### Automatic reboots

Some updates (e.g. kernel upgrades) require a reboot. This can be automated by adding the following.

```bash
Unattended-Upgrade::Automatic-Reboot "true";
Unattended-Upgrade::Automatic-Reboot-Time "04:00";
```

### APT timers and schedules

I will configure the timers for `apt update` and `apt upgrade` to make sure they run within my maintanence window.

`apt update` timer

```bash
sudo systemctl edit apt-daily.timer
sudo systemctl restart apt-daily.timer
```

`apt upgrade` timer

```bash
sudo systemctl edit apt-daily-upgrade.timer
sudo systemctl restart apt-daily-upgrade.timer
```

The following is an example configuration to have the timer set to run daily at a random time between 1:10-1:20AM.

:warning: Make sure `apt update` runs before `apt upgrade` if you configure a randomized delay.

```bash
[Timer]
OnCalendar=
OnCalendar=01:10
RandomizedDelaySec=600
```

If you want to change the schedule of `apt update`, `autocelean` or `upgrade` you can do this by editing the following file.

```bash
sudo nano /etc/apt/apt.conf.d/20auto-upgrades
```

```bash
// How often (in days) to apt update
APT::Periodic::Update-Package-Lists "1";

// How often (in days) to download new packages
APT::Periodic::Download-Upgradeable-Packages "1";

// How often (in days) to clean the apt clean
APT::Periodic::AutocleanInterval "7";

// How often (in days) to run unattended-upgrades
APT::Periodic::Unattended-Upgrade "1";
```

## Mail notifications

To get email notifications, I use [msmtp](https://marlam.de/msmtp/) — a lightweight, sendmail-compatible SMTP client (and the successor to ssmtp).

### Installation

```bash
sudo apt update && sudo apt install msmtp msmtp-mta
```

### Configuration

Switch to root - `su - root`.

Create the config file `/etc/msmtprc`.

```bash
cat << 'EOF' > /etc/msmtprc
defaults
tls on

account example.com
auth on
host mail.example.com
port 25
user username
from someone@example.com
passwordeval "cat /etc/msmtp-pass"

account default : example.com
EOF
```

Store the password in a separate file.

```bash
cat << 'EOF' > /etc/msmtp-pass
password
EOF
```

Set the correct permissions.

```bash
chmod 600 /etc/msmtp-pass
chmod 600 /etc/msmtprc
```

Finally, I will configure `unattended-upgrades` to send mail notifications.

```bash
sudo nano /etc/apt/apt.conf.d/52unattended-upgrades-local
```

```bash
Unattended-Upgrade::Mail "recipient@example.com";
Unattended-Upgrade::MailReport "on-change";
Unattended-Upgrade::Sender "root@host";
```

## Dryrun and enable

Before wrapping up, make sure everything is working by enabling the service and doing a dry run.

```bash
sudo systemctl enable --now unattended-upgrades && sudo unattended-upgrades -d --dry-run
```

Finally, try a manual run once to confirm

```bash
sudo unattended-upgrades -d
```

## Summary

With `UnattendedUpgrades`, my debian server now takes care of itself:

+ Security and feature updates are installed automatically  
+ Old and unused packages are cleaned up  
+ Reboots happen during the night when needed  
+ I get notified by email if something goes wrong  

This setup saves me from manually running updates every couple of weeks and reduces the risk of forgetting critical security patches. For production environments, you might want to limit it to security updates only — but for home labs and small servers, full unattended upgrades are a great balance between convenience and safety.
