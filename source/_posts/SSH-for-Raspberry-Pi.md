---
title: SSH for Raspberry Pi
date: 2025-02-02 12:02:29
tags:
- DIY
- Raspberry Pi
- Computer Science
---

## Introduction
I recently purchased a Cana Kit that came with a Raspberry Pi 5. I plan to host this blog on the Pi and use it for some other things like a Pihole server and a Pi VPN. In this post, I'll explain how I set up my Raspberry Pi 5 to securely SSH into it from the same network. In subsequent posts, I'll explain how to do this from outside the network by using a VPN. While [Raspberry Pi Connect](https://www.raspberrypi.com/software/connect/) exists now, I generally prefer doing the security myself because I'm a software engineer. Here were my high level goals for this project:
- Be able to SSH into my Pi.
- Change the default SSH port (22). Since I'll be hosting a web server on this Pi, my router's IP will be publicly exposed, which makes it a target for automated scanning tools to try to crack into my Raspberry Pi with brute force attacks. Although I will keep SSH restricted to the internal network only, it never hurts to be extra safe.
- Add a firewall to only expose the SSH port (until I start hosting this site).
- Use public-private key (i.e. key pair) authentication for sign-in and turn off password authentication.

<!-- more -->

## Steps
In broad strokes, here are the steps:
1. Enable SSH on the Pi. This, by default, is disabled.
2. Change the default SSH port by modifying the SSH server's configuration.
3. Install [Uncomplicated Firewall](https://wiki.ubuntu.com/UncomplicatedFirewall) and only allow traffic to the SSH port.
4. Test SSH from my laptop, accounting for the fact that my router uses DHCP.
5. Create a key-pair on my laptop and copy the public key to my Pi.
6. Modify the SSH settings on my Pi to restrict authentication to only key pair authentication.

### 1. Enable SSH
I followed the instuctions [here](https://www.raspberrypi.com/documentation/computers/remote-access.html#enable-the-ssh-server) to enable SSH. If you're trying to follow along and have trouble, try running `sudo raspi-config` from the Terminal of your Pi to access settings.

### 2. Change the Default SSH Port
I picked a random available port that wasn't port 22, which is the default SSH port. I opened the SSH configuration file `/etc/ssh/sshd_config` (NOTE: `/etc/ssh/sshd_config` is the SSH server settings while `/etc/ssh/ssh_config` is the SSH client settings) and modified the `Port` line. I used the basic text editor `nano` to do this.
{% codeblock lang:bash line_number:false %}
sudo nano /etc/ssh/sshd_config
# I changed this line to my custom SSH port
Port 12345
# I saved and exited, then restarted ssh using the command below
service ssh restart
{% endcodeblock %}

### 3. Install and Configure ufw
Once again at the terminal, I installed `ufw` using `apt-get`.
{% codeblock lang:bash line_number:false %}
sudo apt-get install ufw
sudo ufw allow 12345
sudo ufw enable
{% endcodeblock %}

This is all it takes to run a simple firewall! It's that simple.

### 4. Test SSH
So this is where things got a little interesting. On an internal network, the router effectively acts as a network controller and manages a subnet. There are billions of IOT (Internet of Things) devices in today's age, and a basic IP address only has 2^32 bits available. Simple math will tell you that's only 4 billion IP addresses. Therefore, we use subnets to allow for internal networks to have their own address space (only accessible within the network) while the router itself has a single public IP exposed to the wider network (often times, the Internet).

For example, let's say my private IP address is 10.0.0.1 and my subnet mask is 255.255.255.0 (a standard /24 subnet). If you do a logical AND of these two values, you'll get 10.0.0.0. This is the first available address on my subnet! If I take the inversion of the subnet (0.0.0.255) and add it to the first available address, I get the last available address on the subnet. Typically, this is reserved as the broadcast address and cannot be used. In this example, the broadcast address is 10.0.0.255.

Now, my router uses Dynamic Host Configuration Protocol (DHCP), which dynamically assigns IP addresses to the devices in my network. This means that the private IP address of my Raspberry Pi can change at pretty much anytime. It might be 10.0.0.2 right now, but it could be 10.0.0.100 tomorrow. Therefore, I can't simply rely on `ssh piusername@<static IP>` to access my Pi. Given the MAC address of a device is constant, I used this fact to find the IP address associated with my Pi's MAC address. Firstly, to find the MAC address, I ran this command

{% codeblock lang:bash line_number:false %}
ip link show eth0
{% endcodeblock %}

I then tasked Chat GPT with writing a simple script to locate my Pi on my internal network and then SSH into it. Here is that script with some modifications from myself:

{% codeblock ssh_pi.sh lang:bash line_number:false %}
#!/bin/bash

# Replace with your Raspberry Pis MAC address (format: xx:xx:xx:xx:xx:xx)
PI_MAC="b8:27:eb:xx:xx:xx"

# Find the IP address using arp
PI_IP=$(arp -an | awk -v mac="$PI_MAC" '$0 ~ mac {gsub(/[()]/, "", $2); print $2}')

# If arp fails, use nmap. Make sure to replace the IP with your router's CIDR address
if [[ -z "$PI_IP" ]]; then
    echo "Raspberry Pi not found on the network. Trying network scan..."
    # Use nmap as a fallback
    PI_IP=$(nmap -sn 10.0.0.1/24 | grep -B 2 "$PI_MAC" | head -1 | awk '{print $5}')
fi

if [[ -z "$PI_IP" ]]; then
    echo "Could not find Raspberry Pi. Ensure it's powered on and connected."
    exit 1
fi

echo "Found Raspberry Pi at $PI_IP"
echo "Connecting via SSH..."

# Replace piusername with your username
ssh piusername@"$PI_IP" -p 12345
{% endcodeblock %}

I then saved it, made it executable, and added an alias to my `.zshrc` file.
{% codeblock lang:bash line_number:false %}
alias sshpi="~/dev/ssh_pi.sh"
{% endcodeblock %}

Now all I have to do is type in `sshpi` in my laptop terminal to SSH into my Pi!

### 5. Use Key Pair Authentication for SSH
Finally, I changed the authentication mode to only key pair authentication. I generated an Ed25519 Key Pair using the following command:
{% codeblock lang:bash line_number:false %}
ssh-keygen -t ed25519
{% endcodeblock %}

Why Ed25519 over RSA? [Check out this article](https://www.brandonchecketts.com/archives/its-2023-you-should-be-using-an-ed25519-ssh-key-and-other-current-best-practices)...

After that simply copy over the public key to your Pi. Grab the IP address output in Step 4 (if it's changed in the last 5 minutes, then simply run the script above again to find the new IP). In the example below, I will use 10.0.0.2.

{% codeblock lang:bash line_number:false %}
ssh-keygen -t ed25519
ssh-copy-id -p 12345 -i ~/.ssh/id_ed25519.pub piusername@10.0.0.2
{% endcodeblock %}

### 6. Restrict SSH Auth to Key Pair Authentication
Lastly, after SSHing into your Pi, modify the SSH configuration again and restart the service.

{% codeblock lang:bash line_number:false %}
sudo nano /etc/ssh/sshd_config

# Disable other forms of authentication
PasswordAuthentication no
ChallengeResponseAuthentication no
KbdInteractiveAuthentication no
UsePAM no

# Disable root login for extra security
PermitRootLogin no

# Explicitly enable public key authentication
PubkeyAuthentication yes

# I saved and exited, then restarted ssh using the command below
service ssh restart
{% endcodeblock %}

## Conclusion
That's it! Next up, I'll be working on actually migrating my blog to be hosted on this Pi and writing about it.