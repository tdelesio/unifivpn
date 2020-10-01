# Unifi VPN
Setup A Unifi VPN Network and route outbound traffic to it

#Configure VLAN
Create a new LAN network.  Set the VLAN to 2

#Configure WiFi SSID
Create a new WiFi Network.  Make sure it belongs to the VPN network created.

#Configure USG
usg.config

#Assign Switch/Port to VLAN


#Setup Second NIC
configure linux routing
vi /etc/network/interfaces
add

auto enp3s0
iface enp3s0 inet dhcp

# The loopback network interface
auto lo
iface lo inet loopback
auto enp2s0
iface enp2s0 inet static
    address 10.0.0.10
    netmask 255.255.0.0
auto enp3s0
iface enp3s0 inet static
    address 192.168.2.5
    netmask 255.255.255.0
    gateway 192.168.2.1

#Change Default gateway
sudo route delete default gw 10.0.0.1 enp2s0
sudo route add default gw 192.168.2.1 enp3s0


#Testing
1.  curl -s http://ifconfig.co/json | jq
2.  Inbox Traffic
3.  IP check from rest of network

#TODO
1.  persist default GW
2.  persist usg configs on GUI changes
