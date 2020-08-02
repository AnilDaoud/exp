---
layout: post
title:  "PF and SSHGuard on Mac OS X"
tags: experiments pf sshguard macosx
---

Quick memo about setting up PF and SSHGuard on Mac OS X, all tested on Catalina (10.15.6) in August 2020.
This is mostly for me to have all the info in one place.

## PF on Mac OS X

The reference article online is [here](https://manjusri.ucsc.edu/2015/03/10/PF-on-Mac-OS-X/)

PF (Packet Filter) is OpenBSD’s system for filtering TCP/IP traffic and doing Network Address Translation. PF in OS X, however, appears to be based on the FreeBSD port of PF. Like FreeBSD 9.X and later, OS X appears to use the same version of PF as OpenBSD 4.5.

The main PF configuration file is */etc/pf.conf*

Start pf with
```sh
$ sudo pfctl -E
```
Check status
```sh
$ sudo pfctl -s info
```

Reload /etc/pf.conf:
```sh
$ sudo pfctl -f /etc/pf.conf
```

## Protecting SSHD without SSHGuard

Append the following lines to /etc/pf.conf (see [Section 30.3.3.5 - Using Overload Tables to Protect SSH](https://www.freebsd.org/doc/handbook/firewalls-pf.html) of FreeBSD Handbook for an explanation):

```
table <bruteforce> persist
block quick from <bruteforce>
pass in inet proto tcp to any port ssh \
    flags S/SA keep state \
    (max-src-conn 5, max-src-conn-rate 5/5, \
     overload <bruteforce> flush global)
```

Reload /etc/pf.conf:
```sh
$ sudo pfctl -f /etc/pf.conf
```
Over time, the table bruteforce will be filled by overload rules and its size will grow incrementally, taking up more memory. We can expire table entries using pfctl. For example, this command will remove bruteforce table entries which have not been referenced for a day (86400 seconds):
```sh
$ sudo pfctl -t bruteforce -T expire 86400
```

Then you can automate this command daily in a cronjob or in a timed job. Or you can use SSHGuard

## Protecting SSHD with SSHGuard

Reference article [here](https://smallthingsfloat.com/2013/08/09/configure-sshguard-for-os-x-10-8/)

sshguard is a daemon that protects SSH and other services against brute-force attacks, similar to fail2ban. sshguard is different from the latter in that it is written in C, is lighter and simpler to use with fewer features while performing its core function equally well. On Mac OS X, install it with homebrew:

```sh
$ brew install sshguard
$ sudo brew services start sshguard
```

Add the following lines to */etc/pf.conf*:

```
#############
# Variables #
#############
ext_if="en0"
wifi="en1"
loop_if="lo0"
############
# SSHGuard #
############
table <sshguard> persist
block in quick on $ext_if proto tcp from <sshguard> to any port 22 label "sshguard"
block in quick on $wifi proto tcp from <sshguard> to any port 22 label "sshguard"
```

Reload /etc/pf.conf:
```sh
$ sudo pfctl -f /etc/pf.conf
```

See it in action:
```sh
sudo pfctl -T show -t sshguard
```

