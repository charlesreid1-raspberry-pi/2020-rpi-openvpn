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
    * [Wifi Access Point](#wifi-access-point)
    * [Network Bridge](#network-bridge)
    * [Testing](#testing)

# Summary

* The raspberry pi has two (or one) wifi cards plugged in
* The pi accesses the internet via one of the wifi cards connected to an internet hotspot
* The pi sets up an encrypted VPN tunnel with a VPN server (not included, see [this guide](https://github.com/charlesreid1/2020-openvpn-mfa-google-auth))
* The pi creates a wifi access point, and runs the DHCP/DNS software required to run the network
* The pi bridges the access point network with the VPN network, and sends all external traffic
  out through the tunnel, instead of going through the pi's internet hotspot
* All internal traffic on the wifi hotspot is routed by the pi, all external traffic is handled
  by the VPN server

## Network Architecture

* Client wifi network - Pi is connected as a client to an internet-connected wifi
  network (or just plug the Pi into an ethernet wall jack)
* VPN network - Pi uses client wifi network connection to connect to an OpenVPN server
  and set up an encrypted tunnel. All external traffic from the Pi is directed through
  this VPN tunnel.
* Access point wifi network - Pi broadcasts an AP name and clients can join the network.
  Pi runs necessary DHCP and DNS server software to be able to set up the network.
* Network bridge - the VPN tunnel and the AP wifi network are bridged, so that external
  traffic flows through the VPN tunnel instead of through the Pi's client wifi connection.

# Step-by-Step Guide

## Hardware

## Connecting to the Pi

## Pi Internet Connection

## OpenVPN Client

## Wifi Access Point

## Network Bridge

## Testing

