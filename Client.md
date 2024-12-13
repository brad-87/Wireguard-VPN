The easiest way to get local network access with Wireguard is to use something like pfSense of OPNSense as your home router. However there is another way, by making a VPN gateway. 
This can be any computer on the network you want to access. However, a RaspberryPI running a Debian based OS is more than capable of this task.
***
First off, we will install a few required programs.

So, login as non-sudo user. Then enter the root shell
```bash
sudo -i
```

Update all packages on the system and install some new programs
```bash
apt update && apt upgrade -y
apt install wireguard iftop net-tools htop -y
```


***
We need to turn on packet forwarding

```bash
nano /etc/ufw/sysctl.conf
```

Uncomment this line. You may need to reboot after this, I'm not 100% sure
```bash
net.ipv4.ip_forward=1
```

***
***
On the VPN server, you run the auto add script, copy that output and paste it into this file. Save and exit.
```bash
nano /etc/wireguard/wg0.conf
```
It will look some thing like this:
```bash
[Interface]
PrivateKey = ***************************************
Address = 10.99.99.2/32
MTU = 1420

[Peer]
PublicKey = ***************************************
AllowedIPs = 10.99.99.1/24
Endpoint = [VPN SRV IP]:50000
PersistentKeepalive = 20
```
***
***
Then we need to activate the masquerade rules in IP Tables.
Clear IP Tables first to ensure default settings:
```bash
iptables -F && iptables -X && iptables -P INPUT ACCEPT && iptables -P FORWARD ACCEPT && iptables -P OUTPUT ACCEPT
```
Then we will add the forwarding and masquerade rules. This will essentially turn your Pi into a router.
```bash
iptables -A INPUT -i wg0 -j ACCEPT
iptables -A FORWARD -i wg0 -j ACCEPT
iptables -A FORWARD -i wg0 -o eth0 -m state --state RELATED,ESTABLISHED -j ACCEPT
iptables -A FORWARD -i eth0 -o wg0 -m state --state RELATED,ESTABLISHED -j ACCEPT
iptables -A OUTPUT -o wg0 -j ACCEPT

iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

```
***
So the main problem with IP Tables is that if you reboot the device, you loose the settings. Let's fix that
```bash
apt install iptables-persistent
iptables-save > /etc/iptables/rules.v4
```
***
***
Now, in theory, this should work. So generate another config from the server, install Wireguard on your phone and give it a test. The tool 'iftop' can be very handy when diagnosing networking issues
```

    WARRANTY VOID IF STICKER REMOVED

              ,d88b.d88b,
              88888888888
              `Y8888888Y'
                `Y888Y'
                  `Y'

```
