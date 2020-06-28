# 2020-rpi-openvpn-hostapd

Notes on setting up a Raspberry Pi to create a wifi AP and tunnel all traffic through OpenVPN.

# Table of Contents

* [Summary](#summary)
    * [Network Architecture](#network-architecture)
* [Step\-by\-Step Guide](#step-by-step-guide)
    * [Hardware](#hardware)
    * [Connecting to the Pi](#connecting-to-the-pi)
    * [Pi Internet Connection](#pi-internet-connection)
    * [OpenVPN Client](#openvpn-client)
    * [Testing](#testing)

# Summary

* The raspberry pi has one wifi card plugged in
* The pi accesses the internet via the wifi card connected to an internet hotspot
* The pi sets up an encrypted VPN tunnel with a VPN server (not included, see [this guide](https://github.com/charlesreid1/2020-openvpn-mfa-google-auth))
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

## Installing Software

Once you have your network connection working,
install some software:

```
sudo apt-get -y install openvpn
```

## OpenVPN Client

To set up the OpenVPN client, obtain your OpenVPN profile file (`*.ovpn`) and your server certificate
file (`*.ca`). We illustrate using Private Internet Access, a third party VPN provider, as an example.

The commands below should be run by the root user.

### OpenVPN Profile Setup

Obtain the client certificate and profile files:

```
cd /tmp
wget https://www.privateinternetaccess.com/openvpn/openvpn.zip
unzip openvpn.zip
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
sed -i 's+^auth-user-pass+& /etc/openvpn/login' /etc/openvpn/${PROFILE}.ovpn
sed -i 's+^ca ca.rsa.2048.crt+& /etc/openvpn/ca.rsa.2048.crt' /etc/openvpn/${PROFILE}.ovpn
sed -i 's+^crl-verif crl.rsa.2048.pem+& /etc/openvpn/crl.rsa.2048.pem' /etc/openvpn/${PROFILE}.ovpn
```

### Test that it works

Test the VPN connection:

Run `curl -4 icanhazip.com` before and after you bring the VPN up to verify your IP has changed:

```
openvpn --config /etc/openvpn/${PROFILE}.conf
```

Use `curl -6 icanhazip.com` to test whether your IPv6 address has changed.

### Configure iptables

Modify the iptables on the Pi as follows:

```
# Allow loopback device (internal communication)
sudo iptables -A INPUT -i lo -j ACCEPT
sudo iptables -A OUTPUT -o lo -j ACCEPT

# Allow all local traffic.
sudo iptables -A INPUT -s 192.168.0.0/24 -j ACCEPT
sudo iptables -A OUTPUT -d 192.168.0.0/24 -j ACCEPT

# Allow VPN establishment
# (Port 53 not necessary if you connect using an IP instead of a domain name)
sudo iptables -A OUTPUT -p udp --dport 53 -j ACCEPT
sudo iptables -A INPUT -p udp --sport 53 -j ACCEPT
sudo iptables -A OUTPUT -p udp --dport 1198 -j ACCEPT
sudo iptables -A INPUT -p udp --sport 1198 -j ACCEPT

# Accept all connections through VPN
sudo iptables -A OUTPUT -o tun+ -j ACCEPT
sudo iptables -A INPUT -i tun+ -j ACCEPT

# Set default policies to drop all communication unless specifically allowed
sudo iptables -P INPUT DROP
sudo iptables -P OUTPUT DROP
sudo iptables -P FORWARD DROP
```

Make these persistent:

```
sudo apt-get -y install iptables-persistent
sudo netfilter-persistent save
sudo systemctl enable netfilter-persistent
```

Enable this VPN client as a service that will start up on boot:

```
systemctl enable openvpn@${PROFILE}
```

and finally, **reboot the Pi**.

