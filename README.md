# Pi-hole with PiVPN (WireGuard) on AWS

# Table of Contents
* [Create a VPS](#create-a-VPS)
* [Install Pi-hole](#Install-Pi-hole)
* [Install and setup VPN](#Install-and-setup-your-VPN)
* [Install Unbound](#Install-Unbound)
* [Notes and Troubleshooting](#Notes)

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

# Install Pi-hole

## Step 2: Connect to VPS

`ssh -i /Users/[your user]/.ssh/PiVPNHOLE.pem ubuntu@[your host]` 

## Step 3: Install Pi-hole

`curl -sSL https://install.pi-hole.net | bash`
* Take note of your Pi-hole's web interface IP and the password

# Install and Setup VPN

## Step 4: Install PiVPN
`curl -L https://install.pivpn.io | bash`

## Step 5: Create Profiles

`pivpn add -n [config name]`
*  where [config name] is a unique name for each of your devices (e.g., mphone, mlaptop). You can repeat this step for as many devices that you want to connect to your Pi-hole.

## Step 6: Setup Split-Tunneling
`sudo nano /etc/wireguard/configs/[config name].conf`
* Change `AllowedIPs` from `"0.0.0.0/0, ::0"` to `"[Pi-hole IP address]/32, [DNS IP]/32"`. 
	* The [DNS IP] is listed in [interface] and by default `10.6.0.1/32`.
	* `[Pi-hole IP address]/32` allows access to the Pi-hole web interface and `[DNS IP]/32` allows DNS requests 
* Note split-tunneling only routes the DNS (i.e., Pi-hole ad-blocking) vs., all of your data through your VPN which will save bandwidth to keep you on the free tier.
* You'll need to repeat this for each [config name] you created in step 5.

## Step 7: Display QR code to Transfer Profile to Mobile
`pivpn -qr [config-name.conf]`

## Step 8: Transfer Profiles from VPS to Local

`scp -i ~/.ssh/PiVPNHOLE.pem ubuntu@[your host]:~/configs/[config-name.conf] ~/etc/wireguard/`

# Troubleshooting
* Before being able to remotely log in, I had to run the command `chmod 600 ~/.ssh/PiVPNHOLE.pem`.
* After clicking "generate keys" in PiVPN, you may get `/tmp/setupVars.conf permission denied`. I solved this by deleting that file.
* You may need to run the PiVPN script as sudo. Run with `curl -L https://install.pivpn.io | sudo bash`.

