# amazon-linux-2023-wireguard

This is a guide on how to install a WireGuard server on a fresh Amazon Linux 2023 EC2 instance and connecting clients to it.

## Set up an EC2 instance

Launch a new EC2 instance with Amazon Linux 2023 image on it.

- Instance type `t3.micro` works perfectly well for this purpose.
- Create a new key pair of type `RSA` and `.pem` file format, give it a name and then store it somewhere safe.
- Create a new security group that allows SSH traffic from `0.0.0.0/0` (or a stricter IP range if you want).
- Configure storage with `gp3` type, give it a decent amount of space, like 20 GB.

When instance is launched, decide what port WireGuard should listen on, then go to the instance and add a new inbound rule that allows said port and `UDP` protocol, set source to `0.0.0.0/0` to allow anything from the internet to connect.

In this guide the port is set to `51820`.

## Install and set up dependencies

Always make sure you are a sudo user first, by running:

```shell
sudo -i
```

### Wireguard

```shell
curl -Lo /etc/yum.repos.d/wireguard.repo \
  https://copr.fedorainfracloud.org/coprs/jdoss/wireguard/repo/epel-7/jdoss-wireguard-epel-7.repo
yum clean all

yum install -y wireguard-dkms wireguard-tools
```

Then you'll have to enable wg-quick service and ipv4 forwarding:

```shell
systemctl enable wg-quick@wg0.service
sysctl net.ipv4.conf.all.forwarding=1 | tee -a /etc/sysctl.d/forwarding.conf
```

### Iptables

```shell
yum install iptables-services
iptables-save | tee /etc/sysconfig/iptables
systemctl enable iptables
```

## Generate server keys

```shell
cd /etc/wireguard
wg genkey | tee privatekey | wg pubkey | tee publickey
```

You'll now have a private and public key generated that you'll need to use later.

## Set up server config

Edit addresses and port to your liking. `ens5` is the default network interface on Amazon Linux 2023. Find the interface with `ip -c a` command (NOT loopback).

```shell
mkdir -p /etc/wireguard/
cd /etc/wireguard
touch wg0.conf
```

Set `wg0.conf` content to:

```
[Interface]
Address = 10.106.28.1/32
SaveConfig = true
PostUp = iptables -A FORWARD -i %i -j ACCEPT; iptables -t nat -A POSTROUTING -o ens5 -j MASQUERADE
PostDown = iptables -D FORWARD -i %i -j ACCEPT; iptables -t nat -D POSTROUTING -o ens5 -j MASQUERADE
ListenPort = 51820
PrivateKey = SERVER_PRIVATE_KEY

[Peer]
PublicKey = CLIENT_PUBLIC_KEY
AllowedIPs = 10.106.28.2/32
```

Then restart the instance with `reboot` command.

### Port forwarding

If you want to forward specific ports to a client, then you can add the following to `wg0.conf`:

```
PostUp = iptables -t nat -A PREROUTING -i ens5 -p tcp --dport 35220 -j DNAT --to-destination 10.106.28.2
PostDown = iptables -t nat -D PREROUTING -i ens5 -p tcp --dport 35220 -j DNAT --to-destination 10.106.28.2
```

Where `35220` is the port you want to forward, and `10.106.28.2` is the client.

After editing you need to restart the wg0 service by running `systemctl restart wg-quick@wg0.service`.

## Set up client config

```
[Interface]
PrivateKey = CLIENT_PRIVATE_KEY
Address = 10.106.28.2/32

[Peer]
PublicKey = SERVER_PUBLIC_KEY
AllowedIPs = 0.0.0.0/1, 128.0.0.0/1
Endpoint = EC2_PUBLIC_IP:SERVER_PORT
```

Set `AllowedIPs = 0.0.0.0/0` if you want to enable kill-switch. This config allows you to communicate to LAN devices.

You'll find `EC2_PUBLIC_IP` on the EC2 instance page in the AWS console.

Set `SERVER_PORT` to `ListenPort` value in server config.

## Allow new clients

```shell
sudo wg set wg0 peer CLIENT_PUBLIC_KEY allowed-ips CLIENT_IP
```

If you use Windows and have the official WireGuard client then `CLIENT_PUBLIC_KEY` will be automatically generated for you.

Replace `CLIENT_IP` with e.g. `10.106.28.2`.
