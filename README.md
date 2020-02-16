# Transmission
[Transmission](https://transmissionbt.com/) BitTorrent client
can be set up as a web interface that is accessible from any other
machine in the same LAN.

## Install required packages:
```bash
dnf install transmission-cli transmission-common transmission-daemon
```

## Edit configuration file
The configuration file is located at:
`/var/lib/transmission/.config/transmission-daemon/settings.json`

Set download directory:
```json
"download-dir": "/data/torrents",
```

Set the user name and password:
```json
"rpc-password": "transmission",
"rpc-username": "transmission",
```

Give access to any device on LAN:
```json
"rpc-whitelist": "127.0.0.1,192.168.50.*",
```

Allow other users to access *Transmission* files:
```json
"umask": 2,
```

## Fix send/receive buffer issue
In its default setup, accessing *Transmission Web Interface* will fail with a buffer error. To fix this, create
the file `/etc/sysctl.d/99-transmission-daemon.conf` with two
lines:
```bash
net.core.rmem_max=4194304
net.core.wmem_max=1048576
```
Load new *systemctl* configuration file:
```bash
sysctl --load=/etc/sysctl.d/99-transmission-daemon.conf
```

## Configure firewall
The ports used by *Transmission* are blocked by a firewall by default. These can be opened using *firewalld*.

Create a new *firewalld service* by copying the default configuration for *Transmission*:
```bash
cp /usr/lib/firewalld/services/transmission-client.xml /etc/firewalld/services/transmission-web.xml
```

Edit the new file by adding port 9091, which is used by *Transmission Web Interface*:
```xml
<port protocol="tcp" port="9091"/>
```

Edit the file `/etc/firewalld/zones/FedoraServer.xml` by adding
the new service:
```xml
<service name="transmission-web"/>
```

Reload *firewalld* to apply the new settings:
```bash
firewall-cmd --reload
```

## Start *Transmission* daemon automatically
Start transmission daemon with the new settings:
```bash
systemctl start transmission-daemon
```
Load automatically at boot time:
```bash
systemctl enable transmission-daemon
```