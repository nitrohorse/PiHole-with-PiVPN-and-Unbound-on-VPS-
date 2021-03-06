# [Pi-hole](https://pi-hole.net/) with [PiVPN (WireGuard)](https://pivpn.io/) on [AWS](https://aws.amazon.com/free)

# Table of Contents
* [Create a VPS](#Create-a-VPS)
* [Install Pi-hole](#Install-Pi-hole)
* [Install and Setup PiVPN](#Install-and-Setup-PiVPN)
* [Troubleshooting](#Troubleshooting)

# Create a VPS
## Step 1: Create a free Ubuntu server in AWS

* Create an [AWS account](https://aws.amazon.com/free/).
* Go to EC2, click "Launch Instance", select "free tier" and choose Ubuntu.
* Manually configure, and click through each step until you get to Security groups, and add the following:
	* Custom UDP
	* Port Range: `51820`
	* Source: `0.0.0.0/0`
	* Description: WireGuard
* Download your new keypair (e.g. `PiVPNHOLE.pem`) and save it to ~/.ssh.
* In your EC2 terminal, note your "PublicDNS (IPv4)," it'll look like: `ec2-###.location.com`. I call this [your host] below.
* Click "Elastic IP" to create an Elastic IP, then click "actions" and "associate", and associate the Elastic IP to your new server.

**Note:** If you are planning to use this as a VPN (no split-tunneling), use LightSail. AWS has a £0.12 / gb cost on outbound transfers. This means that if you use 1tb / month, you'll spend £120. If you use Lightsail, the £3.50 tier gets you 1tb / month which saves you £116.50.

## Optional

One way to avoid filling up your Pi-hole logs with queries to your VPS' ["Private DNS"](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-dns.html), e.g. `[hostname].[region].compute.internal`, is to disable `systemd-resolved`.

Disable `systemd-resolved`.
```
sudo systemctl disable systemd-resolved
sudo systemctl stop systemd-resolved
```

Delete the symlink `/etc/resolv.conf`.
```
sudo rm /etc/resolv.conf
```

Recreate `/etc/resolv.conf`.
```
sudo vim /etc/resolv.conf
```

Add a nameserver which the VPS will use for resolving network requests (such as updating the Pi-hole blocklists or packages).
```
nameserver 9.9.9.9 # Quad9
```

Add the hostname (found in `/etc/hostname`) to `/etc/hosts` (e.g. `sudo vim /etc/hosts`):
```
127.0.0.1 localhost [hostname]
```

# Install Pi-hole

## Step 2: Connect to VPS

`ssh -i /Users/[your user]/.ssh/PiVPNHOLE.pem ubuntu@[your host]`

## Step 3: Install Pi-hole

`curl -sSL https://install.pi-hole.net | bash`
* Take note of your Pi-hole's web interface IP and the password

# Install and Setup PiVPN

## Step 4: Install PiVPN
`curl -L https://install.pivpn.io | bash`

## Step 5: Create Profiles

`pivpn add -n [profile-name]`

## Step 6: Setup Split-Tunneling
`sudo vim /etc/wireguard/configs/[profile].conf`

* Change `AllowedIPs` from `"0.0.0.0/0, ::0"` to `"[Pi-hole IP address]/32, [DNS IP]/32"`. 
	* The `[DNS IP]` is listed under `Interface` and by default is `10.6.0.1/32`.
	* `[Pi-hole IP address]/32` allows access to the Pi-hole web interface and `[DNS IP]/32` allows DNS requests .

## Step 7: Display QR code to Transfer Profile to Mobile
`pivpn -qr [profile-name]`

## Step 8: Transfer Profiles from VPS to Local

`scp -i ~/.ssh/PiVPNHOLE.pem ubuntu@[your host]:/etc/wireguard/configs/[profile-name.conf] ~/etc/wireguard/`

## Optional: Setup Encrypted DNS

Setup encrypted DNS with several options:
- DNS over HTTPS with [Cloudflared](https://docs.pi-hole.net/guides/dns/cloudflared/).
- DNS over HTTPS or DNSCrypt with [dnscrypt-proxy](https://github.com/DNSCrypt/dnscrypt-proxy/wiki/Installation).
- Your own recursive resolver or DNS over TLS with [Unbound](https://docs.pi-hole.net/guides/dns/unbound/).

# Troubleshooting
* Before being able to remotely log in, I had to run the command `chmod 600 ~/.ssh/PiVPNHOLE.pem`.
* After clicking "generate keys" in PiVPN, you may get `/tmp/setupVars.conf permission denied`. I solved this by deleting that file.
* You may need to run the PiVPN script as sudo. Run with `curl -L https://install.pivpn.io | sudo bash`.
