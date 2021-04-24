# Port Redirection

## Introduction

In this page we will explain how to redirect an incoming connection on a specific port (e.g: `69/UDP`), 
to another port (e.g: `6969/UDP`), so that you can have an application listening on an unprivileged port
whilst still mapping it to a privileged port thanks to `iptables(8)`.

## Diagram

<center style="padding: 10px; background-color: #FFF;">
    <img src="figures/port-redirection-1.svg" alt="Port Redirection Diagram"/>
</center>


## Pre-Requisites

- iptables

## Steps

Given:
- The incoming port (client-facing port), called `$INCOMING_PORT`, in this case `69/UDP`
- The port on which the service on your local machine is listening to, called `$SERVICE_PORT` in this case `6969/UDP`
- The protocol of your service / incoming port, called `$PROTOCOL`, in this case `UDP`

The redirection is performed with:

```bash
sudo iptables -t nat -I PREROUTING -i enp8s0 -p "$PROTOCOL" --dport "$INCOMING_PORT" -j REDIRECT --to-port "$SERVICE_PORT"
```

... and don't forget to open the ports!

```bash
sudo iptables -I INPUT -p "$PROTOCOL" --dport "$INCOMING_PORT" -j ACCEPT
sudo iptables -I INPUT -p "$PROTOCOL" --dport "$SERVICE_PORT" -j ACCEPT

```
