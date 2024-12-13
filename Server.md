This is intended to assist in the creation of a remote access Wireguard VPN server on a Debian based VPS.
This VPN can be used for remote access to private networks. AKA Your home network
During this process, as a 'newbie' with networking, I did learn quite a lot, however, the pain was real.


1. Create a Debian based VPS. Try not to forget the password, Preeny.
2. Ensure it has a static IP.
3. Set the system time-zone and time.
4. Secure the bloody thing.
	1. Disable SSH root login
	2. Setup SSH key-pair auth
	3. Disable SSH password auth
	4. Make a non-sudo user to access the server
5. Install and configure unattended-upgrades
6. Set the system time-zone and time because you already skipped a step.
7. Install a firewall. UFW is used in this setup
8. Consider installing fail2ban


***
***
First off, we will install a few required programs.

So, login as non-sudo user. Then enter the root shell
```bash
su -
```

Update all packages on the system and install some new programs
```bash
apt update && apt upgrade -y
apt install wireguard iftop net-tools htop ufw -y
```

***
We need to modify some things before the firewall gets enabled.
***

```bash
nano /etc/default/ufw
```

Change from deny to accept
```bash
DEFAULT_FORWARD_POLICY="ACCEPT"
```

***

```bash
nano /etc/ufw/sysctl.conf
```

Uncomment this line
```bash
net.ipv4.ip_forward=1
```

***

```bash
nano /etc/ufw/before.rules
```

Just above the filter rules add this chunk

```bash	
######################### NAT #########################
#NAT table rules
*nat
:POSTROUTING ACCEPT [0:0]
-A POSTROUTING -o eth0 -j MASQUERADE
COMMIT
####################### END NAT #######################	
```

***
***

Now we add rules to the firewall and enable it
```bash
ufw allow [SSH listen port]/tcp
ufw allow [Wireguard listen port]/udp
ufw enable
```
Enable any other rules (eg. 80,443) if applicable.

***
Hopefully you didn't get kicked out...
***

Lets make the encryption keys that Wireguard will use.
```bash
mkdir /etc/wireguard/keys
cd /etc/wireguard/keys
wg genkey | tee privatekey | wg pubkey > publickey
```
***
Create a config file for Wireguard interface. This will define the IP's for the overlay network and listening port. There will be more going into this file later on. The #1 is important, don't ignore it!

```bash
nano /etc/wireguard/wg0.conf
```

Paste the following into the config

```bash
[Interface]
Address = 10.99.99.1/24
ListenPort = 50000
PrivateKey = [server private key]
#1
```
***

Enable the Wireguard interface
```bash
systemctl enable --now wg-quick@wg0
wg
```
Use the 'wg' command to check the status of the interface
***
***
Adding a peer to the interface config manually is tough. Don't do it, use my script.

Make a file for the script
```bash
touch /root/auto-add.sh
chmod +x /root/auto-add.sh
nano /root/auto-add.sh
```
Then go ahead and paste this into the file, save and exit.
```bash
#!/bin/bash
# Scritpt to generate a peer config file, and update the wireguard interface config file

############################################################################################
############################################################################################
######################################### TUNABLES #########################################
############################################################################################
############################################################################################

# Wireguard config location and filename
conf_file="/etc/wireguard/wg0.conf"

# Wireguard server keys directory and filename
keys_dir="/etc/wireguard/keys/publickey"

# First 3 octets of the ip range used by wireguard
vpn_ip_range="10.99.99"

# Endpoint address:port
endpoint="[vps-ip]:[wireguard listen port]"

############################################################################################
############################################################################################
############################################################################################
############################################################################################


# Ask for a name
echo "Name for connection (no spaces, please):"
read varname

# Get the last line of the wg interface config file
varip=$( tail -1 $conf_file )

# Remove the first character of the last line to get the last used ip
varip="${varip:1}"

# Add 1 to the last used ip for the ip to use here
varip=$(($varip + 1))

# Generate keys and load into variables
wg genkey | tee privatekey | wg pubkey > publickey
varprivate=$( cat privatekey )
varpublic=$( cat publickey )
varserverpublic=$( cat $keys_dir )

# Write peer config file
echo "Making user config"
echo "[Interface]"                                              >>      wg-$varname.conf
echo "Address = $vpn_ip_range.$varip/32"                        >>      wg-$varname.conf
echo "PrivateKey = $varprivate"                                 >>      wg-$varname.conf
echo "[Peer]"                                                   >>      wg-$varname.conf
echo "AllowedIPs = $vpn_ip_range.0/24"                          >>      wg-$varname.conf
echo "PublicKey = $varserverpublic"                             >>      wg-$varname.conf
echo "Endpoint = $endpoint"                                     >>      wg-$varname.conf
echo "PersistentKeepAlive = 20"                                 >>      wg-$varname.conf

# Add peer to server config
echo "Adding peer to server config"
echo "[Peer]"                                                   >>      $conf_file
echo "# Connection for $varname"                                >>      $conf_file
echo "AllowedIPs = $vpn_ip_range.$varip/32"                     >>      $conf_file
echo "PublicKey = $varpublic"                                   >>      $conf_file

#make a note of the ip we just used so this script can read it next time
echo "#$varip"                                                  >>      $conf_file

#cleanup waste files
rm privatekey
rm publickey

# Display file contents
echo "#####################################################"
cat wg-$varname.conf
echo "#####################################################"
echo "        Copy above config to local computer"
echo "#####################################################"

# Uncomment to auto delete generated files
#rm wg-$varname.conf
```
Running this file will generate a config for the connecting device, and automatically add the required config to the interface config. Note there is no auto-remove script. If you stuff up, open the interface config and delete what it added
***

After any change to the config, you need to restart Wireguard for it to take effect. Resart with:
```bash
wg-quick down wg0
wg-quick up wg0
```

***
***
***

Once you have added a peer, for remote access to a private network, you need to add the allowed IP's  to the interface config.
```bash
[Interface]
Address = 10.99.99.1/24
ListenPort = 50000
PrivateKey = [server private key]

[Peer]
# Connection From JimBob
AllowedIPs = 10.99.99.2/32, 192.168.1.1/24
PublicKey = [peer public key]
```
In the peer section, you can see that the allowed IPs list the network that will be routable over the VPN. In this case the IPs 192.168.1.1 - 255 will be routable.
Note that you can't have the same IP range on multiple peers. So If you want to use this as a site-to-site VPN you will need to plan the IP ranges on the provate networks.
Remember: After any change to the config, you need to restart Wireguard for it to take effect.












