# Background
The configuration detailed in this repo is what I use on an old laptop that was converted into
a *CentOS* server. I currently use *CentOS Stream 8*, which provides stability but also uses a
*rolling release* model.

# EPEL
Extra Packages for Enterprise Linux (EPEL) contains many useful packages that are not available in the default CentOS repos.

Install it via DNF:
```bash
dnf install epel-release
```

# Transmission
[Transmission](https://transmissionbt.com/) BitTorrent client
can be set up as a web interface that is accessible from any other
machine in the same LAN.

## Install required packages:
```bash
dnf install transmission-cli transmission-common transmission-daemon
```

## Generate configuration files
Start and stop the *transmission daemon* in order to generate the default configuration files:
```bash
systemctl start transmission-daemon
systemctl stop transmission-daemon
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

## Allow access to torrent folder
The *Transmission* web interfece requires write permissions to the `/data/torrents` folder.
The optimal way to accomplish this is to give ownership of the folder to the user *transmission*,
created automatically at installation.

Run the following command to assign ownership and permissions for the *torrent* folder:
```bash
chown transmission:transmission /data/torrents
chmod 0755 /data/torrents
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

Find the default *firewalld* zone:
```bash
firewall-cmd --get-default-zone
```

Add add the new *transmission-web* service to the default zone (*FerdoraServer* in this example):
```bash
firewall-cmd --zone=FedoraServer --add-service=transmission-web --permanent
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

## Configure SELinux
Each directory shared via *Samba* must be given a SELinux context. To do so for the directory `/data`, run the command:
```bash
chcon -R -t samba_share_t /data
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
        browseable = yes
        writeable = yes
        hide unreadable = yes
```

## Restart services
Restart *Samba* services to apply configuration:
```bash
systemctl restart smb nmb
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