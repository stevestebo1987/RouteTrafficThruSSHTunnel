# RouteTrafficThruSSHTunnel

Routing plex traffic through an SSH tunnel

This guide creates a reverse SSH tunnel to route all Plex server traffic through it.

Step 2 is done on the tunnel, all other steps are done on the plex server.
1. Setup SSH keys (if you already have key based authenthication setup skip to step 2)

On plex server:

1a. Create SSH key

root@ubuntu:~# ssh-keygen -t rsa
Generating public/private rsa key pair.
Enter file in which to save the key (/root/.ssh/id_rsa):
Created directory '/root/.ssh'.
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /root/.ssh/id_rsa.
Your public key has been saved in /root/.ssh/id_rsa.pub.

Passaphrase must be empy for autossh to work!

1b. Copy SSH key

root@ubuntu:~# ssh-copy-id root@TUNNELIP
/usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed
/usr/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed -- if you are prompted now it is to install the new keys
root@TUNNELIP's password:

Number of key(s) added: 1

Now try logging into the machine, with:   "ssh 'root@TUNNELIP'"
and check to make sure that only the key(s) you wanted were added.

1c. Connect to tunnel

root@ubuntu:~$ ssh root@TUNNELIP
Welcome to Ubuntu 16.04.1 LTS (GNU/Linux 4.9.7-x86_64-linode80 x86_64)
Last login: Wed Feb 22 03:49:58 2017
root@ubuntu:~#

You should not be promted for a password
2. Edit tunnel's SSH server configuration

2a. Add "Gatewayports yes" to sshd_config

root@ubuntu:~# nano /etc/ssh/sshd_config

Change:

...
Port 22
...

To:

...
Port 22
GatewayPorts yes
...

2b. restart sshd

sudo service ssh restart

3. Install autossh and create systemd service:

3a. Install autossh

sudo apt install autossh

3b. Create systemd service file

sudo nano /etc/systemd/system/autossh-plex-tunnel.service

Contents:

[Unit]
Description=AutoSSH tunnel service Plex on local port 32400
After=network.target

[Service]
Environment="AUTOSSH_GATETIME=0"

ExecStart=/usr/bin/autossh -M 40584 -o "compression=no" -o "cipher=aes128-gcm@openssh.com" -o "ServerAliveInterval 30" -o   "ServerAliveCountMax 3" -NR 32400:localhost:32400 root@TUNNELIP
User=changeme
[Install]
WantedBy=multi-user.target

4. Enable and start service

sudo systemctl enable autossh-plex-tunnel
sudo systemctl start autossh-plex-tunnel

4b. Check SSH tunnel

sudo systemctl status autossh-plex-tunnel

If tunnel was created successfully output should look something like this:

autossh-plex-tunnel.service - AutoSSH tunnel service Plex on local port 32400
   Loaded: loaded (/etc/systemd/system/autossh-plex-tunnel.service; enabled; vendor preset: enabled)
   Active: active (running) since Mon 2017-02-20 03:11:14 CET; 2 days ago
 Main PID: 32570 (autossh)
   CGroup: /system.slice/autossh-plex-tunnel.service
           ├─32570 /usr/lib/autossh/autossh -M 40584 -o compression=no -o cipher=aes128-gcm@openssh.com -o ServerAliveInterval 30 -o ServerAliveCountMax 3 -NR 32400:localhost:32400 root@TUNNELIP
           └─32574 /usr/bin/ssh -L 40584:127.0.0.1:40584 -R 40584:127.0.0.1:40585 -o compression=no -o cipher=aes128-gcm@openssh.com -o ServerAliveInterval 30 -o ServerAliveCountMax 3 -NR 32400:localhost:32400 root@TUNNELIP

Feb 20 03:11:14 Hetzner systemd[1]: Started AutoSSH tunnel service Plex on local port 32400.
Feb 20 03:11:14 Hetzner autossh[32570]: starting ssh (count 1)
Feb 20 03:11:14 Hetzner autossh[32570]: ssh child pid is 32574

go to http://TUNNELIP:32400 on your browser, if it does not load the tunnel was not setup correctly
5. Point plex.tv to correct ip

Plex.TV Web App > Settings > Server > Network > Custom server access URLs

https://TUNNELIP:32400,http://TUNNELIP:32400

6. Only allow local connections to port 32400

sudo iptables -A INPUT -p tcp -s localhost --dport 32400 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 32400 -j DROP
sudo iptables-save > /etc/iptables.rules

7. Make iptables rules apply at startup

edit /etc/network/interfaces

Change

auto  eth0
iface eth0 inet static
  address   xxx.xxx.xxx.xxx

To:

auto  eth0
iface eth0 inet static
 pre-up iptables-restore < /etc/iptables.rules
  address   xxx.xxx.xxx.xxx

Done!
Feel free to leave a comment with your questions or suggestions.
