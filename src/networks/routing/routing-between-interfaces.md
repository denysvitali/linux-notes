# Routing traffic between interfaces

## Introduction

This page explains how to perform routing from one interface (`enp8s0`) to another (`wlp7s0`), so that you can,
for example, share your Wi-Fi connection over ethernet.

<center style="padding: 10px; background-color: #FFF;">
<img src="figures/network-diagram-1.svg" alt="Network Diagram"/>
</center>

## Pre-Requisites

- iptables
- dnsmasq

## Steps

### Configure dnsmasq

Edit `/etc/dnsmasq.conf` and add the following:

```conf
interface=enp8s0
domain-needed
bogus-priv
no-resolv
no-poll
dhcp-range=192.168.2.2,192.168.2.100,72h
dhcp-option=option:router,192.168.2.1
```

Substitute `enp8s0` with your input interface name, and choose a range of your liking:
in this case we're using the `192.168.2.0/24` subnet.

### Add your new IP address

In the previous section we configured `dnsmasq` to provide (via DHCP) an IP address in the range
`192.168.2.2 - 192.168.2.100` and to keep that lease for 72 hours.

We're ready to take over the `192.168.2.1` IP address!

```bash
sudo ip route add 192.168.2.1/24 dev enp8s0
sudo ip addr show dev enp8s0 | grep 'inet[^6]' # Show IPv4 addresses
```

If everything is correct, this will be the output:

```
    inet 192.168.2.1/24 scope global enp8s0
```

We have almost everything in place:
- We have a DHCP server ready to serve our subnet range and the router (our machine)'s IP address
  when asked, on `enp8s0`.
- We have associated the `192.168.2.1` IP address to the interface `enp8s0`

We are still missing:
- Routing the traffic from `enp8s0` to `wlp7s0`

### Route the traffic

#### Enable IP forwarding

To start routing the traffic, we have to enable IP forwarding:

```bash
cat /proc/sys/net/ipv4/ip_forward
echo 1 | sudo tee /proc/sys/net/ipv4/ip_forward
```

To make the change persistant, you can add the `/etc/sysctl.d/ip-forward` file with the following line:
```
net.ipv4.ip_forward = 1
```

Then, to activate the changes run:
```
sudo sysctl -p
```

#### Define iptables rules

```bash
# NAT the traffic from 192.168.2.0/24 to wlp7s0 (internet)
sudo iptables -t nat -A POSTROUTING -o wlp7s0 -s 192.168.2.0/24 -j MASQUERADE

# Forward the RELATED,ESTABLISHED traffic from wlp7s0 (internet) back to enp8s0
sudo iptables -A FORWARD -i wlp7s0 -o enp8s0 -m state --state RELATED,ESTABLISHED -j ACCEPT

# Accept traffic coming from enp8s0 and forward it
sudo iptables -A FORWARD -i enp8s0 -o wlp7s0 -j ACCEPT

# Enable traffic for DHCP
sudo iptables -I INPUT -p udp -i enp8s0 --dport 67 -j ACCEPT
iptables -t filter -A INPUT -i enp8s0 -p udp -s 0.0.0.0 --sport 68 -d 255.255.255.255 --dport 67 -j ACCEPT
iptables -t filter -A INPUT -i enp8s0 -p udp -s 192.168.2.0/24 --sport 68 -d 192.168.2.1 --dport 67 -j ACCEPT
```


### Start the services 

```bash
sudo systemctl start dnsmasq
```