# Internet sharing with VPN on RPi

Here are instructions on how to setup a computer (as a Raspberry Pi) to act as a VPN sharing device. It connects to a WIFI network in the environment, connects to a wireguard VPN and shares the VPN connection to its eth0. eth0 acts as a DHCP, so when a device is connected, it accesses the VPN connection.

This requires access to a wireguard VPN server. For example, you could setup your router at home to open a wireguard VPN.

## Set up eth0 DHCP

```
sudo apt install isc-dhcp-server
nano -w /etc/dhcp/dhcpd.conf
```

Paste the following:

```
default-lease-time 600;
max-lease-time 7200;
option subnet-mask 255.255.255.0;
option broadcast-address 192.168.2.255;
option routers 192.168.2.1;
option domain-name-servers 8.8.8.8, 8.8.4.4;
option domain-name "mydomain.example";

subnet 192.168.2.0 netmask 255.255.255.0 {
    range 192.168.2.10 192.168.2.100;
    option routers 192.168.2.1;
    option subnet-mask 255.255.255.0;
    option domain-name-servers 8.8.8.8, 8.8.4.4;
    option broadcast-address 192.168.2.255;
}

option netbios-name-servers 192.168.2.1;
```

Set static ip for eth0:

```
sudo nmcli con add type ethernet ifname eth0 con-name eth0
sudo nmcli con modify eth0 ipv4.addresses 192.168.2.1/24
sudo nmcli con modify eth0 ipv4.method manual
sudo nmcli con modify eth0 connection.autoconnect yes
sudo nmcli con up eth0
```

Make the DHCP server is configured to listen on the correct interface `sudo nano /etc/default/isc-dhcp-server` and set the following: `INTERFACESv4="eth0"`

Restart the DHCP service `sudo systemctl restart isc-dhcp-server`

# Wireguard

```
sudo apt install wireguard
sudo apt install resolvconf
sudo dpkg-reconfigure resolvconf
```

Move the VPN config file to `/etc/wireguard/wg0.conf`

```
sudo systemctl start wg-quick@wg0
sudo systemctl enable wg-quick@wg0
```

## IP forwarding and configuring NAT

Edit the /etc/sysctl.conf file to enable IP forwarding `sudo nano /etc/sysctl.conf`. Uncomment `net.ipv4.ip_forward=1` and apply changes with `sudo sysctl -p`.

Use iptables to set up NAT so that traffic from eth0 can be routed through the WiFi connection:

```
sudo apt install iptables
sudo iptables -t nat -A POSTROUTING -o wg0 -j MASQUERADE
sudo iptables -A FORWARD -i eth0 -o wg0 -j ACCEPT
sudo iptables -A FORWARD -i wg0 -o eth0 -m state --state RELATED,ESTABLISHED -j ACCEPT
```

To ensure these iptables rules persist after a reboot, you need to save them. You can use the iptables-persistent package.

```
sudo apt install iptables-persistent
sudo netfilter-persistent save
```

Restart the DHCP server `sudo systemctl restart isc-dhcp-server`

## Connect to WIFI networks

Scan for networks and list them
```
sudo nmcli -p device wifi rescan
nmcli device wifi list
```

Connect to wifi using BSSID or SSID. BSSID is unique, whereas multiple networks can have the same SSID
```
sudo nmcli -p device wifi connect <wifi_name> --ask
```

Other commands:
```
nmcli device status
nmcli device disconnect <interface>
```

The PI will try to connect to the previous WIFI after a reboot.

## Check public IP
`curl ifconfig.me`
