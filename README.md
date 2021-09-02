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

### Why Pi-hole

From the official web site we read `The Pi-hole® is a DNS sinkhole that protects your devices from unwanted content, without installing any client-side software`, so basically Pi-hole runs in our local network as a DNS resolver and it kills queries for known bad domains and supports DNS-over-HTTP requests.

You can find more on its site [pi-hole.net](https://pi-hole.net)

This guide was create just by reading Pi-hole project documentation [here](https://docs.pi-hole.net)

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

## Common set up

* Set the hostname (replace `<NEW_HOSTNAME>` with the name you desire)

  ```shell
  sed -i "s/$HOSTNAME/<NEW_HOSTNAME>/g" /etc/hostname /etc/hosts
  ```

* Set the timezone

  ```shell
  dpkg-reconfigure tzdata
  ```

* Upgrade the system to the latest version of all packages by running:

  ```shell
  apt update && apt -y upgrade
  ```

* Install iptables

  ```shell
  apt install iptables
  ```

## Set up DNS-over-HTTPS

Before setting up Pi-hole we're going to setup a proxy to translate our DNS queries into DNS-over-HTTPS requests.

There are different ways to achieve that, such as using [DNSCrypt](https://dnscrypt.info/implementations), [cloudflared](https://github.com/cloudflare/cloudflared), amongst others

Here we're going to explain how to implement `DNSCrypt` because it's a more flexible solution and could be used with different DNS providers.

* Install DNSCrypt-proxy

  ```shell
  apt install dnscrypt-proxy
  ```

* Customize the config file

  ```shell
  nano /etc/dnscrypt-proxy/dnscrypt-proxy.toml
  ```

  In `server_names` you can choose any from [this](https://dnscrypt.info/public-servers/) list, I suggest to use `cloudflare`

  ```text
  server_names = ['cloudflare']
  ```

  Since port 53 is going to be used by Pi-hole, in `listen_addresses` line set another port, i.e. 5353

  ```text
  listen_addresses = ['127.0.0.1:5353']
  ```

  Also set the logging directory to `/var/log.hdd` instead

  ```text
  [query_log]
  file = '/var/log.hdd/dnscrypt-proxy/query.log'

  [nx_log]
    file = '/var/log.hdd/dnscrypt-proxy/nx.log'
  ```

  For more information on how to customize it, please refer to the official example [here](https://github.com/DNSCrypt/dnscrypt-proxy/blob/master/dnscrypt-proxy/example-dnscrypt-proxy.toml)

* Config dnscrypt-proxy.socket

By default dnscrypt-proxy.socket uses it's own configuration in the SystemD socker unit. We have to edit this as well to avoid issues with Pi-hole:

```shell
nano /etc/systemd/system/sockets.target.wants/dnscrypt-proxy.socket
```

And set the ListenStream and ListenDatagram to use the correct port:

```text
ListenStream=127.0.2.1:5353
ListenDatagram=127.0.2.1:5353
```

Save the file and move on

* Enable pidfile

  In order to monitor if `dnscrypt-proxy` is running we will need to enable the creation of the pid file by editing the SystemD service unit `/etc/systemd/system/multi-user.target.wants/dnscrypt-proxy.service`:

  ```shell
  nano /etc/systemd/system/multi-user.target.wants/dnscrypt-proxy.service
  ```

  Set `-pidfile` at the end of the `ExecStart` line, like:

  ```text
  ExecStart=/usr/sbin/dnscrypt-proxy -config /etc/dnscrypt-proxy/dnscrypt-proxy.toml -pidfile /var/run/dnscrypt-proxy/dnscrypt-proxy.pid
  ```

  Reload the unit files:

  ```shell
  systemctl daemon-reload
  ```

* Restart the service

  ```shell
  systemctl restart dnscrypt-proxy
  ```

## Pi-hole installation

At the moment of writing this guide, Armbian is not officially supported by Pi-hole, but in fact as Armbian is based on Debian as Raspbian, it should work. So make sure to create the following environment variable to avoid checking the OS:

```shell
export PIHOLE_SKIP_OS_CHECK=true
```

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

Follow the installation steps...

When you see `Select Upstream DNS Provider. To use your own, select Custom.` choose custom and add `127.0.0.1#5353`

Then finish the installation steps.

You should now be able to access Pi-hole webadmin portal at [http://pi.hole/admin](http://pi.hole/admin)

<!-- ### Pi-hole Webadmin over HTTPS

There's one thing that bothers me a lot, it's the fact that Pi-hole web interface uses plain HTTP protocol :facepalm:

In order to fix that we're going to use [CertBot](https://certbot.eff.org) to generate and maintains our [Let's Encrypt](https://letsencrypt.org) certificate and set up Lighttpd.

* Install CertBot

  ```shell
  apt install letsencrypt
  ``` -->

## Set up DHCP server

You can either use the NanoPi as a DHCP for your LAN or your can edit your actual DHCP options to use the NanoPi IP address as the default DNS server.

## Watchdog

### What is a watchdog

A watchdog is an electronic timer used for monitoring hardware and software functionality. The software uses a watchdog timer to detect and recover fatal failures.

### Why using a watchdog

We use a watchdog to make sure we have a functional DNS server. If a problem comes up, the computer should be able to recover itself back to a functional state. We will configure the board to reboot if DNS doesn't respond, or a specific process isn’t running any more.

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
  pid = /var/run/pihole-FTL.pid
  pid = /var/run/dnscrypt-proxy/dnscrypt-proxy.pid
  retry-timeout = 60
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
apt install unattended-upgrades
```

The default set up of this package installs security updates for the current release. If you want to update all packages when available, take a look at the `/etc/apt/apt.conf.d/50unattended-upgrades`.

To test the package behaviour, we can run:
```shell
unattended-upgrade --debug --dry-run
```

## Logrotate

Armbian in NanoPi has the logs located in two directories. The first is a ramdisk (`/var/log/`) which is usually around 50MB size. This is definitely not enough to keep our logs for more than a week, and depending on how much connection we have a day will not even hold 24h of logs before you start getting errors such as:

```shell
cannot write to log file '/var/log/xxx.log': No space left on device
```

The second one is located in the root partition (`/var/log.hdd/`).

The good practice here would be to save all logs in the disk, or at least safekeeping a compressed copy in the disk for security.

### Armbian ram log

Let's start by increasing the `/var/log` ramdisk from 50MB to 100MB.

* Edit `/etc/default/armbian-ramlog` file

  ```shell
  nano /etc/default/armbian-ramlog
  ```

* Set `SIZE` to 100M

* Apply the changes

  ```shell
  systemctl restart armbian-ramlog.service
  ```

### Pi-hole logs

Our main log generator is going to be Pi-hole.

By default Pi-hole stores all logs in `/var/log/pihole*` but as we already know that location is inside the ramdisk on Armbian. So, we have to move them to another location such as `/var/log.hdd/`

* Edit Pi-hole-FTL config file

  ```shell
  nano /etc/pihole/pihole-FTL.conf
  ```

* Set log files location

  ```text
  LOGFILE=/var/log.hdd/pihole-FTL.log
  ```

* Restart the service

  ```shell
  systemctl restart pihole-FTL.service
  ```

### DNSmasq logs

* Edit dnsmasq config file for Pi-hole

  ```shell
  nano /etc/dnsmasq.d/01-pihole.conf
  ```

* Change log facility from `log-facility=/var/log/pihole.log` to `log-facility=/var/log.hdd/pihole.log`

* Edit Pi-hole logrotate config file

  ```shell
  nano /etc/pihole/logrotate
  ```

* Change the path inside the file from `/var/log` to `/var/log.hdd`, save and exit.

Pi-hole will start writing and rotating to the new log files automatically.

```Keep in mind tho, that on every update of Pi-hole this configuration will be reset to the default.```

### Armbian logrotate

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
/var/log/*.log /var/log/*/*.log {
  daily
  rotate 0
  create
  missingok
}
```

What this config is going to do is rotate all log files in `/var/log/` as well their child directories.

This can be tested by:

```shell
logrotate -d /etc/logrotate.d/nanopi
```

The `-d` flag will list each log file it is considering to rotate.

As Logrotate is set up to run daily via Cron we don't have to do any further change.

## SSH hardening

These are just some good practices to hardening the SSH daemon.

* Generate a SSH key on your workstation to connect to NanoPi using SSH certificate

  ```shell
  ssh-keygen -t rsa -N '' -f ~/.ssh/<NANOPI_HOSTNAME>
  ```

* Copy the content of `~/.ssh/<NANOPI_HOSTNAME>.pub` into NanoPi `~/.ssh/authorized_keys`

* Set your SSH config to use the certificate when connecting to NanoPi by adding a new entry in `~/.ssh/config`

  ```shell
  Host nanopineo3
    HostName <IP>
    Port <SSH_PORT>
    User <REMOTE_USER>
    IdentityFile ~/.ssh/<NANOPI_HOSTNAME>
  ```

* Edit SSH daemon config file `/etc/ssh/sshd_config` with the following lines:

  ```shell
  # Disable root login
  PermitRootLogin no

  # Disable password authentication
  ChallengeResponseAuthentication no
  PasswordAuthentication no
  ```

* Apply the previous config, just restart the SSH daemon:

  ```shell
  systemctl restart ssh
  ```

## Clean the system

* Disable unused SystemD services

  ```shell
  systemctl stop wpa_supplicant systemd-rfkill.service systemd-rfkill.socket hostapd
  systemctl disable wpa_supplicant systemd-rfkill.service systemd-rfkill.socket hostapd
  ```
