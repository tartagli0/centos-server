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
In its default setup, accessing *Transmission Web Interface* will fail with a buffer error. To
fix this, create the file `/etc/sysctl.d/99-transmission-daemon.conf` with two lines:
```bash
net.core.rmem_max=4194304
net.core.wmem_max=1048576
```
Load new *systemctl* configuration file:
```bash
sysctl --load=/etc/sysctl.d/99-transmission-daemon.conf
```

## Configure firewall
The ports used by *Transmission* are blocked by a firewall by default. These can be opened using
*firewalld*.

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

# Samba
This section describes the steps required to create a *samba* share that can
be mapped as a network drive from any *Windows* PC on the same LAN.

Install *samba* package:
```bash
dnf install samba
```

## Create new Samba user
To access the *samba* share, a *Windows* client must log in with a user ID
that exists in the *Fedora* system. In this case, the user name is
*smbguest*.

Create the new user with no *home* directory, as it is not
needed:
```bash
useradd -M smbguest
```

Set password for new user:
```bash
passwd smbguest
```

Set a *samba* password for *smbguest*:
```bash
smbpasswd -a smbguest
```

## Set permissions for shared folders
In order to allow *smbguest* read/write access to a shared folder
(e.g., *myfolder*), the owner must be set to *smbguest* and permissions for
*myfolder* should be set to *755*.

```bash
chown smbguest myfolder
chmod 755 myfolder
```

Files within this folder that belong to another user cannot be deleted
(assuming permissions are properly set) and will not be visible to *smbguest*
if read access is not granted by owner. See 

## Open firewall ports
The required ports for *Samba* are blocked by default and must be opened via
*firewalld*.

*Samba* already exists as a
pre-configured service in *firewalld*, which can be added to
the default firewall zone:
```bash
firewall-cmd --permanent --zone=FedoraServer --add-service samba
```

Reload firewall confguration:
```bash
firewall-cmd --reload
```

## Edit configuration file

The *smb.conf* file is used to configure *Samba* and can contain countless
options. In Fedora systems, this file is located at `/etc/samba/smb.conf`.

The configuration below is all that is needed for a *Samba* server that can
only be accessed from a LAN machine, requires a username and password, and
hides files that a user cannot access.

```
[global]
        workgroup = WORKGROUP
        server string = Samba Server Version %v
        interfaces = lo eth0 192.168.50.204/24
        hosts allow = 127. 192.168.50.
        logging = systemd
        security = user
        passdb backend = tdbsam

[Shared]
        comment = Shared Storage
        path = /data
        writeable = yes
        hide unreadable = yes
```

# Laptop Lid
By default, CentOS will hibernate or sleep when the laptop lid is closed. To prevent this behavior, open the file `/etc/systemd/logind.conf` and add, un-comment, or modify the line:
```bash
HandleLidSwitch=ignore
```
Restart the service with the command:
```bash
systemctl restart systemd-logind.service
```
The laptop lid can now be closed without interrupting the server.