# How to set up Pi-hole to secure your DNS queries with DNS-over-HTTPS and domain filters

This is a step by step guide on how to set up Pi-hole on any linux device to secure your DNS queries via DNS-over-HTTPS and ban all blacklisted domains.

For this example I'm going to use (and test) the set up in a [Nano Pi Neo3](https://www.friendlyarm.com/index.php?route=product/product&product_id=279)

## Requirements

* 1x NanoPi
* 1x MicroSD
* [Armbian distro](https://www.armbian.com/nanopineo3/)

## Considerations

### Why DNS-over-HTTPS

Long story short, DNS protocol uses plain text. Any DNS servers that are contacted plus any routers on the path to those DNS servers would be able to figure out which sites you’re visiting. As you can imagine this could be easily tracked, logged and even modified. Here’s where DNS-over-HTTPS comes into play enabling point to point encryption from the our DNS server to the destination DNS, nobody in between can (easily :D) snoop on or even spoof your web traffic.

### Why Pi-Hole

From the official web site we read `The Pi-hole® is a DNS sinkhole that protects your devices from unwanted content, without installing any client-side software`, so basically Pi-Hole runs in our local network as a DNS resolver and it kills queries for known bad domains and supports DNS-over-HTTP requests.

You can find more on its site [pi-hole.net](https://pi-hole.net)

## OS Installation

### Copy OS to SD Card

I will document the process with Etcher to write Armbian image into SD cards.

* Download [Etcher](https://www.etcher.io/)
* Insert the SD Card in the computer
* Use Ether to write the Armbian image you should have already downloaded (if not check the requirements section for the link)
* Once done, eject the card and insert it in the NanoPi R2S

### Booting NanoPi R2S

* Plug the ethernet cable
* Plug the power cable
* Armbian uses DHCP by default so once you know the IP address assigned from your DHCP server you can SSH into it
* SSH into the box by `ssh root@<IP>` and the default password is `1234`
* Immediately after login the first time it will ask the user to change `root` password and create a new regular user account

## Setup DNS-over-HTTPS

## PiHole installation

For those who want to get started quickly and conveniently may install Pi-hole using the following command:

```shell
curl -sSL https://install.pi-hole.net | bash
```

If, on the other side, you want to have a copy of the installer script you can run:

```shell
curl --output basic-install.sh https://install.pi-hole.net
chmod +x basic-install.sh
sudo ./basic-install.sh
```

## Set up DHCP server

## Watchdog

### What is a watchdog

A watchdog is an electronic timer used for monitoring hardware and software functionality. The software uses a watchdog timer to detect and recover fatal failures.

### Why using a watchdog

We use a watchdog to make sure we have a functional VPN. If a problem comes up, the computer should be able to recover itself back to a functional state. We will configure the board to reboot if WireGuard link is down for too long, or a specific process isn’t running any more.

### Setup the watchdog software

* Install the watchdog software

  ```shell
  apt install watchdog
  ```

* Configure the watchdog to monitor WireGuard network

  ```shell
  nano /etc/watchdog.conf
  ```

  Edit the following lines:

  ```text
  log-dir = /var/log.hdd/watchdog

  interface = wg0

  ping = <REMOTE_WG_IP>

  retry-timeout = 300

  interval = 30
  ```

* Enable and start the service

  ```shell
  systemctl stop watchdog
  systemctl enable watchdog
  systemctl start watchdog
  ```

  ## Unattended security updates

Security updates are crucial to keep our system safe from threats. Even tho we don't have so many services open to the world, one bug is enough to allow attackers to break into our system.

```shell
apt install -y unattended-upgrades
```

The default set up of this package installs security updates for the current release. If you want to update all packages when available, take a look at the `/etc/apt/apt.conf.d/50unattended-upgrades`.

To test the package behaviour, we can run:
```shell
unattended-upgrade --debug --dry-run
```

## Log rotate

Armbian in NanoPi has the logs located in two directories. The first is a ramdisk (`/var/log/`) which is usually around 50MB size. This is definitely not enough to keep our logs for more than a week, and depending on how much connection we have a day will not even hold 24h of logs before you start getting errors such as:

```shell
cannot write to log file '/var/log/xxx.log': No space left on device
```

The second one is located in the root partition (`/var/log.hdd/`).

The good practice here would be to save all logs in the disk, or at least safekeeping a compressed copy in the disk for security.

But if you're using this at home and you don't care much about them apart from realtime debugging when errors happen, then you can basically discard all logs after a day using `logrotate` :)

Let's start by increasing the `/var/log` ramdisk from 50MB to 100MB.

Edit `/etc/default/armbian-ramlog` and set `SIZE` to 100M.

apply the changes by running `systemctl restart armbian-ramlog.service`

Now, let's move to Logrotate. The main config file is located at `/etc/logrotate.conf` and then all sort of directory specific Logrotate definitions inside `/etc/logrotate.d`, let's first edit the default behaviour by:

```shell
nano /etc/logrotate.conf
```

Replace the content of the file for this:
```config
# rotate log files daily
daily

# Old versions are removed
rotate 0

# create new (empty) log files after rotating old ones
create

# uncomment this if you want your log files compressed
compress

# packages drop log rotation information into this directory
include /etc/logrotate.d
```

Now let's see what we have inside `/etc/logrotate.d/` directory:

```shell
ls /etc/logrotate.d/
alternatives  apt  armbian-hardware-monitor  btmp  chrony  dpkg  rsyslog  wtmp
```

And what I'm going to do here is delete everything and create a new config file called `nanopi`. So let's remove everything:

```shell
rm /etc/logrotate.d/*
```

And now let's create the new config file at `/etc/logrotate.d/nanopi` with the following content:

```config
/var/log.hdd/*.log /var/log.hdd/*/*.log {
  daily
  rotate 0
  create
  missingok
}

/var/log/*.log /var/log/*/*.log {
  daily
  rotate 0
  create
  missingok
}
```

What this config is going to do is rotate all log files in `/var/log/` and `/var/log.hdd` as well their child directories.

This can be tested by:

```shell
logrotate -d /etc/logrotate.d/nanopi
```

The `-d` flag will list each log file it is considering to rotate.

As Logrotate is set up to run daily via Cron we don't have to do any further change.

## SSH hardening

These are just some good practices to hardening our SSH daemons, especially when they are publically available.

Add those lines somewhere inside the `/etc/ssh/sshd_config` file:

```shell
# Disable root login
PermitRootLogin no

# Disable password authentication
ChallengeResponseAuthentication no
PasswordAuthentication no

# Limit daemon to only listen on localhost (only for WireGuard client when we enable reverse SSH)
ListenAddress ::1
ListenAddress 127.0.0.1
```

To apply the previous config, just restart the SSH daemon:

```shell
systemctl restart ssh
```