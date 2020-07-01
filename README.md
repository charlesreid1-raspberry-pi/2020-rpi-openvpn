# 2020-rpi-openvpn-hostapd

Notes on setting up a Raspberry Pi to create a wifi AP and tunnel all traffic through OpenVPN.

This document has multiple versions, each version is a different branch.

* [pia-tunnel-only](https://github.com/charlesreid1-raspberry-pi/2020-rpi-openvpn-hostapd/tree/pia-tunnel-only): set up a VPN tunnel with PIA that routes all external traffic through the tunnel

# Table of Contents

* [Summary](#summary)
    * [Network Architecture](#network-architecture)
* [Step 1: VPN Tunnel](#step-1-vpn-tunnel)
    * [Hardware](#hardware)
    * [Pi Network Configuration](#pi-network-configuration)
    * [OpenVPN Client](#openvpn-client)
        * [OpenVPN Profile Setup](#openvpn-profile-setup)
        * [Update the Startup File](#update-the-startup-file)
        * [Test that it works](#test-that-it-works)
        * [Enable VPN Service](#enable-vpn-service)
    * [How It Works](#how-it-works)
* [Step 2: Wifi AP](#step-2-wifi-ap)


# Summary

* The raspberry pi has raspbian buster installed
* The raspberry pi has one wifi card plugged in
* The pi accesses the internet via the wifi card connected to an internet hotspot
* The pi sets up an encrypted VPN tunnel with a VPN server (VPN server not included, see [this guide](https://github.com/charlesreid1/2020-openvpn-mfa-google-auth) for setting one up)
* All external traffic is handled by the VPN server

We cover the setup in 3 steps:

* Establish VPN tunnel
* Create wifi AP
* Bridge wifi AP with VPN tunnel


## Network Architecture

* Client wifi network - pi is connected as a client to an internet-connected wifi
  network (or just plug the pi into an ethernet wall jack)
* VPN network - pi uses client wifi network connection to connect to an OpenVPN server
  and set up an encrypted tunnel. All external traffic from the pi is directed through
  this VPN tunnel.


# Step 1: VPN Tunnel


## Hardware

* Raspberry Pi
* 1-2 wifi dongles
* Ethernet cable (optional)


## Pi Network Configuration

Easiest way to connect to the pi is to edit the filesystem partition of the SD card, specifically edit the `wpa_supplicant`
configuration file to automatically connect to the wifi network providing internet, and you can skip the next step
because the pi will already have an internet connection.

Help the pi automatically connect to a WPA network by editing `/etc/wpa_supplicant/wpa_supplicant.conf`:

```
ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
update_config=1

network={
    ssid="<ssid-network-name>"
    proto=RSN
    key_mgmt=WPA-PSK
    pairwise=CCMP TKIP
    group=CCMP TKIP
    psk="<ssid-network-passphrase>"
}
```

Add this to /etc/rc.local to reset the interface and connect to wifi on boot:

```
sleep 3
ifdown wlan0
sleep 3
ifup wlan0
sleep 3
/sbin/wpa_supplicant   -i wlan0   -P /var/run/wpa_supplicant.wlan0.pid   -D nl80211,wext -c /etc/wpa_supplicant/wpa_supplicant.conf
```

If you are running Raspbian, you should also touch a file named `ssh` in the boot partition of the SD card to start the SSH server.


## OpenVPN Client

To set up the OpenVPN client, obtain your OpenVPN profile file (`*.ovpn`) and your server certificate
file (`*.ca`). We illustrate using Private Internet Access, a third party VPN provider, as an example.

The commands below should be run by the root user.


### OpenVPN Profile Setup

Obtain the client certificate and profile files:

```
cd /tmp
wget https://www.privateinternetaccess.com/openvpn/openvpn.zip
unzip -d openvpn openvpn.zip
cd openvpn
```

This directory contains multiple OpenVPN profiles for each of PIA's server regions. To use them:

```
PROFILE="Belgium"
cp ca.rsa.2048.crt /etc/openvpn/.
cp crl.rsa.2048.pem /etc/openvpn/.
cp ${PROFILE}.ovpn /etc/openvpn/.
```

Now add login credentials to a login file:

```
touch /etc/openvpn/login
echo "USERNAME" >> /etc/openvpn/login
echo "PASSWORD" >> /etc/openvpn/login
```

Finally, modify the configuration file to use this credentials file, and to point to the correct
locations of the certificate and key:

```
sed -i 's+^auth-user-pass+& /etc/openvpn/login+' /etc/openvpn/${PROFILE}.ovpn
sed -i 's+^ca ca.rsa.2048.crt+& /etc/openvpn/ca.rsa.2048.crt+' /etc/openvpn/${PROFILE}.ovpn
sed -i 's+^crl-verif crl.rsa.2048.pem+& /etc/openvpn/crl.rsa.2048.pem+' /etc/openvpn/${PROFILE}.ovpn
```


### Update the Startup File

If you are using an OpenVPN profile (.ovpn) to start the OpenVPN client, run the following command
to use the .ovpn file instead of the .conf file:

```
sed -s 's+\.conf+.ovpn+' /lib/systemd/system/openvpn@.service
```

If you are using a .conf file, do not run this command.


### Test that it works

Test the VPN connection:

Run `curl -4 icanhazip.com` before and after you bring the VPN up to verify your IP has changed:

```
openvpn --config /etc/openvpn/${PROFILE}.ovpn
```

Note that you may have a config file (.conf) instead, in which case, use the config file instead of the .ovpn file.

Use `curl -6 icanhazip.com` to test whether your IPv6 address has changed.


### Enable VPN Service

Enable this VPN client as a service that will start up on boot:

```
systemctl enable openvpn@${PROFILE}
```

and finally, **reboot the pi**.


## How It Works

When a program on the pi sends packets to an IP address, the pi will attempt to reach the
IP address using each network interface (the `traceroute X.X.X.X` command will show the
path the packet will take to its destination).

Suppose the network giving the pi access to the internet is on the CIDR block 192.168.0.0/24.
Further suppose the pi has internet access via a router at 192.168.0.1, and has an IP of
192.168.0.199 assigned to it.

If the pi sends a packet to 192.168.0.200, the traffic leaves the pi via the wireless network
interface, and is sent to the gateway of the 192.168.0.0/24 network, where it is forwarded on
to 192.168.0.200.

If the IP is not on the local network, i.e., anything but 192.168.0.0/24, it will be encrypted
and sent through the tunnel interface that OpenVPN creates.


# Step 2: Wifi AP

In the next section, we cover the setup of a wireless access point (AP) using `hostapd` and
a few other utilities necessary to properly run a wireless network.

We don't cover the bridging until 

The checklist to set up an access point is as follows:

* Configure DNS server (`dhcpd`)
* Configure DHCP server (`isc-dhcp-server`)
* Configure Access Point (`hostapd`)

The network configuration is as follows:

* 192.168.0.0/24 is the home network's CIDR block
    * 192.168.0.1 is the gateway of the home network
    * 192.168.0.199 is the example IP assigned the raspberry pi

* 192.168.10.0/24 is the new wifi network's CIDR block
    * 192.168.10.1 is the gateway, which is the raspberry pi
    * 192.168.10.99 is a client on the new wifi network

## Install Software

Install required software:

```
apt-get update
apt-get -y install udhcpd isc-dhcp-server hostapd
```

## Configure DNS and DHCP

Edit `/etc/dhcpcd.conf` to configure `dhcpcd` on Raspbian Buster.

```
$ cat /etc/dhcpcd.conf
hostname "LAN10"
clientid
persistent
option rapid_commit
option domain_name_servers, domain_name, domain_search, host_name
option classless_static_routes
option interface_mtu
require dhcp_server_identifier
slaac private

# Custom static IP address for wlan1.
interface wlan1
static ip_address=192.168.10.1/24
static routers=192.168.10.1
static domain_name_servers=192.168.10.1
```


## Configure DHCP

Configure the DHCP server in the DHCP server configuration file 
`/etc/default/isc-dhcp-server`:

```
DHCPDv4_CONF=/etc/dhcp/dhcpd.conf
DHCPDv4_PID=/var/run/dhcpd.pid
INTERFACESv4="wlan0"
```

(Ignoring v6 for now to keep it simple.)

## Configure Wireless AP

Configure the access point in the `hostapd` configuration file
`/etc/hostapd/hostapd.conf`:

```
interface=wlan0
wpa=1
ssid=<NAME-OF-NEW-WIFI-AP-HERE>
channel=1
wpa_passphrase=<PASSPHRASE-OF-NEW-WIFI-AP-HERE>
```

