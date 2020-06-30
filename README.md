# 2020-rpi-openvpn-hostapd

Notes on setting up a Raspberry Pi to create a wifi AP and tunnel all traffic through OpenVPN.

This document has multiple versions, each version is a different branch.

* [pia-tunnel-only](https://github.com/charlesreid1-raspberry-pi/2020-rpi-openvpn-hostapd/tree/pia-tunnel-only): set up a VPN tunnel with PIA that routes all external traffic through the tunnel

# Table of Contents

* [Summary](#summary)
    * [Network Architecture](#network-architecture)
* [Step\-by\-Step Guide](#step-by-step-guide)
    * [Hardware](#hardware)
    * [Pi Network Configuration](#pi-network-configuration)
    * [Installing Software](#installing-software)
    * [OpenVPN Client](#openvpn-client)
        * [OpenVPN Profile Setup](#openvpn-profile-setup)
        * [Update the Startup File](#update-the-startup-file)
        * [Test that it works](#test-that-it-works)
        * [Enable VPN Service](#enable-vpn-service

# Summary

* The raspberry pi has one wifi card plugged in
* The pi accesses the internet via the wifi card connected to an internet hotspot
* The pi sets up an encrypted VPN tunnel with a VPN server (VPN server not included, see [this guide](https://github.com/charlesreid1/2020-openvpn-mfa-google-auth) for setting one up)
* All external traffic is handled by the VPN server

## Network Architecture

* Client wifi network - Pi is connected as a client to an internet-connected wifi
  network (or just plug the Pi into an ethernet wall jack)
* VPN network - Pi uses client wifi network connection to connect to an OpenVPN server
  and set up an encrypted tunnel. All external traffic from the Pi is directed through
  this VPN tunnel.

# Step-by-Step Guide

## Hardware

* Raspberry Pi
* 1-2 wifi dongles
* Ethernet cable (optional)

## Pi Network Configuration

Easiest way to connect to the Pi is to edit the filesystem partition of the SD card, edit the `wpa_supplicant`
configuration file to automatically connect to the wifi network providing internet, and you can skip the next step
because the Pi will already have an internet connection.

Create network interfaces in `/etc/network/interfaces`

If you have a wifi card and an ethernet cable:

```
source /etc/network/interfaces.d/*

# loopback
auto lo
iface lo inet loopback

# ethernet - auto
auto eth0
iface eth0 inet dhcp
```

Connect to a WPA network by editing `/etc/wpa_supplicant/wpa_supplicant.conf`:

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

and finally, **reboot the Pi**.

## How It Works

When a program on the pi sends packets to an IP address, the pi will attempt to reach the
IP address using each network interface (the `traceroute X.X.X.X` command will show the
path the packet will take to its destination).

Suppose the network giving the pi access to the internet is on the CIDR block 192.168.0.0/24.
Further suppose the pi has internet access via a router at 192.168.0.1, and has an IP of
192.168.0.199 assigned to it.

If the pi sends a packet to 192.168.0.200, the traffic leaves the pi via the wireless network
interface, and is sent to the gateway of the 192.168.0.0/24 network.

If the IP is not on the local network, i.e., anything but 192.168.0.0/24, it will be encrypted
and sent through the tunnel interface that OpenVPN creates.

