# amazon-linux-2023-wireguard

This is a guide on how to install a WireGuard server on a fresh Amazon Linux 2023 EC2 instance and connecting clients to it.

## Set up an EC2 instance

Launch a new EC2 instance with Amazon Linux 2023 image on it.

- Instance type `t3.micro` works perfectly well for this purpose.
- Create a new key pair of type `RSA` and `.pem` file format, give it a name and then store it somewhere safe.
- In "Network settings" you need to make sure that you automatically assign both an IPv4 and IPv6 address to the instance. For this to work you need to select a VPC and subnet that have IPv6 addresses available.
  - For VPC you need to add an IPv6 CIDR block, easiest is to use an _Amazon-provided IPv6 CIDR block_.
  - The subnet you want to use needs to have said IPv6 CIDR block added to it.
  - Also make sure you have a `::/0` rule added to the _Route table_ of the VPC that targets your _Internet gateway_.
- Create a new security group that allows SSH traffic from `0.0.0.0/0` and `::/0` for IPv6.
  - Add a new security group rule that allows UDP traffic on port `51820` from `0.0.0.0/0` and `::/0`.
- Configure storage with `gp3` type, give it a decent amount of space, like 20 GB.

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

Then you'll have to enable wg-quick service, IPv4 and IPv6 forwarding:

```shell
systemctl enable wg-quick@wg0.service
sysctl net.ipv4.conf.all.forwarding=1 | tee -a /etc/sysctl.d/forwarding.conf
sysctl net.ipv6.conf.all.forwarding=1 | tee -a /etc/sysctl.d/forwarding.conf
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
Address = 10.106.28.1/32, fd42:42:42::1/128
PostUp = iptables -A FORWARD -i %i -j ACCEPT; iptables -t nat -A POSTROUTING -o ens5 -j MASQUERADE
PostUp = ip6tables -A FORWARD -i %i -j ACCEPT; ip6tables -t nat -A POSTROUTING -o ens5 -j MASQUERADE
PostDown = iptables -D FORWARD -i %i -j ACCEPT; iptables -t nat -D POSTROUTING -o ens5 -j MASQUERADE
PostDown = ip6tables -D FORWARD -i %i -j ACCEPT; ip6tables -t nat -D POSTROUTING -o ens5 -j MASQUERADE
ListenPort = 51820
PrivateKey = SERVER_PRIVATE_KEY

[Peer]
PublicKey = CLIENT_PUBLIC_KEY
AllowedIPs = 10.106.28.2/32, fd42:42:42::2/128
```

Then restart the instance with `reboot` command.

### Port forwarding

If you want to forward specific ports to a client, then you can add the following to `wg0.conf`:

```
PostUp = iptables -t nat -A PREROUTING -i ens5 -p tcp --dport 35220 -j DNAT --to-destination 10.106.28.2
PostUp = ip6tables -t nat -A PREROUTING -i ens5 -p tcp --dport 35220 -j DNAT --to-destination fd42:42:42::2
PostDown = iptables -t nat -D PREROUTING -i ens5 -p tcp --dport 35220 -j DNAT --to-destination 10.106.28.2
PostDown = ip6tables -t nat -D PREROUTING -i ens5 -p tcp --dport 35220 -j DNAT --to-destination fd42:42:42::2
```

Where `35220` is the port you want to forward, `10.106.28.2` is the IPv4 client and `fd42:42:42::2` is the IPv6 equivalency.

After editing you need to restart the wg0 service by running `systemctl restart wg-quick@wg0.service`.

Remember to add a new inbound rule to the EC2 instance for every new port.

## Set up client config

```
[Interface]
PrivateKey = CLIENT_PRIVATE_KEY
Address = 10.106.28.X/32, fd42:42:42::X/128
DNS = 1.1.1.1, 1.0.0.1, 2606:4700:4700::1111, 2606:4700:4700::1001

[Peer]
PublicKey = SERVER_PUBLIC_KEY
AllowedIPs = 0.0.0.0/1, 128.0.0.0/1, ::/1, 8000::/1
Endpoint = EC2_PUBLIC_IP:SERVER_PORT
```

Replace `X` with a unique number across all existing clients, starting from 2.

Set `AllowedIPs = 0.0.0.0/0, ::/0` if you want to enable kill-switch. This config allows you to communicate to LAN devices.

You'll find `EC2_PUBLIC_IP` on the EC2 instance page in the AWS console.

Set `SERVER_PORT` to `ListenPort` value in server config.

The `DNS` line consists of Cloudflare's public DNS resolvers.

## Allow new clients

Add new client under the `Peer` header in server config like so:

```
PublicKey = CLIENT_PUBLIC_KEY
AllowedIPs = 10.106.28.X/32, fd42:42:42::X/128
```

Where `X` is the client's unique number.

If you use Windows and have the official WireGuard client then `CLIENT_PUBLIC_KEY` will be automatically generated for you.

Restart service with `systemctl restart wg-quick@wg0.service`.

## Remove clients

Remove client's section under the `Peer` header and restart with `systemctl restart wg-quick@wg0.service`.

## Verification

Always verify your VPN connection with sites like https://ipleak.net/. You should see an IPv4 address and an IPv6 address originating from AWS, as well as some Cloudflare DNS servers.

## References

- https://github.com/alex-ferener/wireguard-on-amazon-linux-2
- https://stanislas.blog/2019/01/how-to-setup-vpn-server-wireguard-nat-ipv6/
- https://github.com/tqml/Wireguard-VPN-EC2-Terraform
