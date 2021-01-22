# Unifi VLAN configuration for VPN Connections

These are the instructions on how to setup your Unifi USG/Cloud Key to configure and connect to a VPN.  The goals are:

* Create a separate network (VLAN) on a different subnet that is separated from the rest of the network
* Create a VPN interface using IPVanish (as the provider) and OpenVPN
* Create a separate SSID/Wifi network that is connected to this VLAN.  Thus any client connecting to the SSID will use the VPN as the outbound connection to the internet.
* Configure a headless docker host with 2 NIC cards.  1 NIC would be connected to the normal LAN and one NIC would be connected to the VLAN.
* Have all outbound traffic on the docker host route over the VPN

## Setup VLAN

We need to create a new network that is split apart from the rest of the networks.  This will have its own subnet and the gateway to the internet will be a VPN connection.  We also want to bind a WIFI network to this new VLAN

1. Connect to the [Unifi portal](https://network.unifi.ui.com/#/controllers/1/50) and launch your instance
1. Go to settings -> networks and create a new network.
1. Set a name for the network and a VLAN id.  I used VPN for the network name and 2 for the VLAN ID for this example.  Set your gateway IP/Subnet.  I used 192.168.2.1/24  Most other options you can leave defaults.  
![LAN](/lan.png)
1. Go to wireless network and create a new wireless network.
1. Set a SSID name and make sure it is part of the network you just created...so VPN here from the dropdown.
![WIFI](/wifi.png)

## Map Switch Port to VLAN

I have a headless linux box that is wired.   If you wanted to connect to the WIFI network, you could skip this step.  Here we need to bind a specific port on the switch to the VPN network.  That way any cat wire you plug into that port will automaticlly talk to the VPN network.

1. Go to devices and find the switch you want to set a port on.
1. Click on the port you want to assign.
1. Click edit selected.  Under Switch Prt Profile select the name(vlan_id) that you want to use.  In my case it says VPN(2).
1. Click apply.
![switch to vpn](/usg_port.png)

## Configure USG

This step assumes you are not going to blindly take my configurations and make them work.  If you are so bold, skip this step as it is not actually required for the end product.  Here is really just the steps you need to work through in order to find the configurations that work for you.  

1. SSH into the USG.  For me it was 10.0.0.1.  Notice the warning when you login.  I didn't...
1. type configure and press enter
1. Execute the usg.config rules one at a time.  
1. type commit;save;exit
1. Test to make sure it works the way you want it to.
1. Assuming it does, export the format to JSON by doing a mca-ctrl -t dump-cfg > con.txt
1. Open up that file and extract out all the things that you changed.  You can see my output in the config.gateway.json file.  
1. I then ran this through a json [formator](https://jsonformatter.curiousconcept.com/) and the output is the formated.config.gateway.json file.  I am honestly not sure if this is needed or not but saw a lot of chatter in the forums about problems with bad formats so I recommend doing it.

[Reference](https://help.ui.com/hc/en-us/articles/215458888-UniFi-USG-Advanced-Configuration-Using-config-gateway-json)

### Configue VPN Connection on USG



## Configure the Cloud Key

So configuring the USG is not persistent.  If you make a change to the GUI at all, all of your configuration changes will be blown away.  That sucks and is not fun removing all the work you have done thus far.  So we need to get to as a perm solution.  

1. SSH into the cloud key.  This IP can be found in the GUI.
1. Move the file you edited (should be named config.gateway.json) to the folder /srv/unifi/data/sites/[site name/default]/
1. If that folder does not exist, the easiest thing to do is to upload an image to the sites.  This will create it for you.
1. Once it is there, execute the command:

```
python -m json.tool config.gateway.json
```
This will return the file if it is corerct.  If it does not you have a format error.
1. Go to the Unifi portal, navigate to the USG and do a force provision.
1. Verify that your changes are now on the USG.  I did that by SSHing into the USG and doing a show interfaces command and having it list out the vpn interface

## Configure Second NIC on Headless Docker Host

The rest of this is probably too specific for anyone else, but I am still going to document it.  Join me if you want...your call.

#Setup Second NIC
1. configure linux routing
```
vi /etc/network/interfaces
```
1. add the lines
```
auto enp3s0
iface enp3s0 inet dhcp
```

1. Verify your interface gets an ip in the right subnet
```
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
```

1. Change Default gateway to point to the new interface
```
sudo route delete default gw 10.0.0.1 enp2s0
sudo route add default gw 192.168.2.1 enp3s0
```

1. Test to make sure it works.
```
curl -s http://ifconfig.co/json | jq
```

#Troubleshooting when things go wrong
VPN servers need to be changed every once in a while.  No idea why...maybe too long on a server.

1. SSH into the gateway (10.0.0.1)
1. cd to
```
/config/user-data
```
1. Change the ovpn file to point to a new server (nyc.ovpn)
1. restart the VPN
```
echo y | reset openvpn interface vtun0
```
