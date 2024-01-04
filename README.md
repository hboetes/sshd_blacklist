# sshd_blacklist
sshd_blacklist is an automated sshd bruteforce blocker, something like fail2ban
or sshguard. But then a lot simpler.

## Before we do anything
First some housekeeping:
- This is beta code!
- All these instructions have to be done as user root.
- Only run this program over an ssh session if you are really careful or you can
  easily get to your system.

## Preparation
I'm running sshd_blacklist on fedora 39. It should be portable to other linux
versions, assuming they have the following dependencies available:

- nftables:    latest linux firewall system, much nicer than iptables.
- logrotate:
- systemd:
- geolookupip: look up the country of origin of an IP.
- zsh:         the shell I prefer to work with.
- ulogd:       logs all firewall activity to a proper logfile, instead of to the
  dmesg, this package is not in the default install, so use my copr, see below


Assuming you also have fedora 39, run the following to install the dependencies:
```
  curl https://copr.fedorainfracloud.org/coprs/hanb/ulogd/repo/fedora-39/hanb-ulogd-fedora-39.repo > /etc/yum.repos.d/hanb-ulogd-fedora-39.repo
  dnf install nftables ulogd logrotate systemd geolookupip zsh
```

Or if you use ubuntu/debian:
```
  apt install nftables ulogd logrotate systemd geolookupip zsh
```

Copy this directory to /etc/sshd_blacklist and go into the new directory:
```
  git clone https://github.com/hboetes/sshd_blacklist/
  cp -r sshd_blacklist /etc
  cd /etc/sshd_blacklist
```

We don't want to lock out ourselves so inspect the whitelist file:
```
  cat whitelist
```

And adapt it to your liking. You can add single IP-addresses and
CIDR-ranges. The smaller this file is the faster the scripts will run.
You can also optimize the whitelist file by putting the most likely
candidates on top.

## Installation
Now let's create the initial blacklist table:
```
  ./sshd_blacklist_init
```

Noone is being blocked yet, but the table is there.
```
  nft list table ip sshd_blacklist | less
```

I like to see all configs in this directory, but that's not where they actually
are loaded so I hardlink them in the right place:
```
  ln sshd_blacklist.nft /etc/nftables/
  ln sshd_blacklist.service /etc/systemd/system/
  ln ulogd.conf /etc/
```

Let's start ulogd and check if it's running:
```
  systemctl enable --now ulogd
  systemctl status ulogd
```
ulogd should be logging all firewall activity to `/var/log/ulogd/ulogd_syslogemu.log`

Now it's getting exiting, we're starting the logwatcher, everyone who is not
whitelisted and tries to log on with a wrong password will get locked out!
```
  systemctl start sshd_blacklist.service
```

If you still have shell acces, you can now enable the logwatcher at boot:
```
  systemctl enable sshd_blacklist.service
```

And with a bit of luck we can see the first brutes being blocked:
```
  tail -n 1000 -F /var/log/ulogd/ulogd_syslogemu.log | grep sshd_blacklist
```

To ensure our rules are loaded at boot inspect `/etc/sysconfig/nftables.conf`.
```
  cat /etc/sysconfig/nftables.conf
```

Which will probably return something like: `include "/etc/nftables/main.nft"`

This means the main nftables config is `/etc/nftables/main.nft`, let's add
`sshd_blacklist.nft` to that file:
```
  echo include /etc/nftables/sshd_blacklist.nft >> /etc/nftables/main.nft
```

Now about expiring bruteforcers, when is that done, and how?  Well... when they
give up trying!  Of course log files get larger over time and they have to be
rotated, so if I look at the log file, just before rotating, and the offender is
no longer there we can safely expire him, which is done by the prerotate script
configured in `ulogd_logrotate`.
```
  ln ulogd_logrotate /etc/logrotate.d/ulog
```

So if you want to expire brutes after an hour change the keyword 'daily' to
'hourly'. Check the logrotate man page for more options.


## Watching the logs:

The main sshd log file and the logwatcher in action:
```
  journalctl -f -u sshd -u sshd_blacklist
```

The firewall log output:
```
  tail -F /var/log/ulogd/ulogd_syslogemu.log
```
