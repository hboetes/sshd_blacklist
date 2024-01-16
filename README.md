# sshd_blacklist
sshd_blacklist is an automated sshd bruteforce blocker, something like fail2ban
or sshguard. But then a lot simpler. We'll let the kernel do the work.

Basically, any ip that is brute-forcing is added to a table which entries will automatically
expire after a given period, but will automatically be renewed after consecutive attempts.

This is what it looks like:
```
Every 2.0: nft list set sshd_blacklist sshd_blacklist|grep expire
                elements = { 43.135.173.175 timeout 1h expires 59m15s557ms, 91.80.156.133 timeout 1h expires 2m52s983ms,
                             95.255.5.73 timeout 1h expires 55m43s572ms, 103.25.47.94 timeout 1h expires 57m22s686ms,
                             103.151.125.131 timeout 1h expires 59m35s656ms, 106.51.80.81 timeout 1h expires 5m41s224ms,
                             112.165.212.156 timeout 1h expires 59m36s215ms, 168.197.49.139 timeout 1h expires 59m18s602ms,
                             200.237.128.234 timeout 1h expires 58m276ms }
```

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
- geoiplookup: look up the country of origin of an IP.
- zsh:         the shell I prefer to work with.

## Install dependencies
Assuming you also have fedora 39, run the following to install the dependencies:
```
  dnf install nftables logrotate GeoIP zsh
```

Or if you use ubuntu/debian:
```
  apt install nftables logrotate geoip-bin zsh
```

### Nice software to have installed:
- ulogd:       logs all firewall activity to a proper logfile, instead of to the
  dmesg.

#### Fedora
This package is not in the default fedora install, but you can use my copr.
I've also added a config file to this repo.

```
  curl https://copr.fedorainfracloud.org/coprs/hanb/ulogd/repo/fedora-39/hanb-ulogd-fedora-39.repo > /etc/yum.repos.d/hanb-ulogd-fedora-39.repo
  dnf install ulogd-pcap
  cp ulogd.conf /etc
  systemctl enable --now ulogd
  cp ulogd_logrotate /etc/logrotate.d/ulogd
```
#### Ubuntu/Debian
Run the following:
```
  apt-get install ulogd2-pcap
  cp ulogd.conf /etc
  systemctl enable --now ulogd
  systemctl status ulogd
```

#### Watching the firewall log:
```
   tail -F /var/log/ulogd/ulogd_syslogemu.log
```


# Installation
Copy this directory to /etc/sshd_blacklist and go into the new directory:
```
  git clone https://github.com/hboetes/sshd_blacklist/
  cp -r sshd_blacklist /etc
  cd /etc/sshd_blacklist
```

We don't want to lock ourselves out, so inspect the whitelist file:
```
  cat whitelist
```

And adapt it to your liking. You can add single IP-addresses and
CIDR-ranges. The smaller this file is, the faster the scripts will run.
You can also optimize the whitelist file by putting the most likely
candidates on top.

Now let's create the initial blacklist table:
```
  ./init_sshd_blacklist
```

Noone is being blocked yet, but the table is there.
```
  nft list table ip sshd_blacklist
```

I like to see all configs in this directory, but that's not where they actually
are loaded so I hardlink them in the right place:
```
  ln sshd_blacklist.nft /etc/nftables/
  ln sshd_blacklist.service /etc/systemd/system/
```

If you also want to set up ulogd, let's configure it and make sure it's up, and running and started at boot:
```
  ln ulogd.conf /etc/
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

And with a bit of luck we can see the first brutes being blocked, with `dmesg -Tw`
or if you have ulogd:
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

## Watching the logs:
The main sshd log and the logwatcher in action, you should see abusers getting
blocked instantly after their attempt.
```
  journalctl -f -u sshd -u sshd_blacklist
```
You can safely ignore messages like: `fatal: Timeout before authentication
for...`, some abusers send multiple requests at the same time, and these got
blocked, so they couldn't finish their attempts.

The firewall log output:
```
  dmesg -Tw
```

Or if you do have ulogd:
```
  tail -F /var/log/ulogd/ulogd_syslogemu.log
```

And let's look at nftables in action:
```
   watch 'nft list ruleset|grep expire'
```

## Expiring with nftables
Here is where the magic is happening:
```
        chain permissiondenied {
                type filter hook input priority filter - 10; policy accept;
                ip saddr @sshd_blacklist update @sshd_blacklist { ip saddr timeout 1h }
                ip saddr @sshd_blacklist log prefix "[sshd_blacklist]" group 0 drop
        }
```
This means: after an abuser has been stopped, every time he tries to log in again, his ban-time will
automatically be reset to 1 hour. Only after giving up he will be "unbanned".


## Disabling sshd_blacklist
In case of emergency, you can disable the sshd_blacklist rules by simply running:
```
   stop_sshd_blacklist
```
