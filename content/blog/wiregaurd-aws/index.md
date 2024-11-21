---
title: Your Own WireGuard VPN Server on AWS 
summary:  Simple guide to deploy your own Wiregaurd VPN server on an EC2 instance in AWS with DDNS.
date: 2024-10-15
cardimage: wiregaurd-full.png
featureimage: wiregaurd.png
caption: ""
authors:
  - bnbhat: author.jpeg

---

## Why a VPN?
As someone from India living in Germany, I’ve often faced challenges accessing Indian websites that work only within India. Whether it’s helping my parents with online tasks or accessing services restricted to India, it can get frustrating. I know many friends and family living abroad face similar issues.

To solve this problem, I decided to deploy my own WireGuard VPN server on AWS. This setup allows me to "pretend" to be in India while staying anywhere in the world. If you’ve ever struggled with geo-restricted content or services, this guide will help you create your own VPN server.

## What is WireGuard?
[WireGuard](https://www.wireguard.com/) is a fast, lightweight, and modern VPN protocol designed for simplicity and security. It encrypts your internet traffic and routes it through a specific server, making it look like you’re accessing the internet from that server’s location. In this case, we’ll configure a server in India.

## AWS and EC2
[AWS (Amazon Web Services)](https://aws.amazon.com/) is a cloud computing platform that offers scalable resources on demand. [EC2 (Elastic Compute Cloud)](https://aws.amazon.com/ec2/) is a service that allows you to run virtual servers (instances) in the cloud. With EC2, you can easily host your WireGuard server.

- New AWS users get access to a t2.micro instance (1GB RAM and 1vCPU) under the free tier for 12 months, which is perfect for this small-scale setup.
- For this deployment, I chose AWS’s Mumbai data center to get an Indian IP address.

## Disclaimer
Before we dive in, please remember:

1. **Use this responsibly**  
Ensure you comply with the terms and conditions of the websites or services you access.  
2. **Understand the costs**  
While AWS offers a free tier, misconfigured setups can lead to unexpected charges. (Yes, those horror stories of massive bills are real!)  
3. **Security is key**  
Keep your VPN configuration private and ensure only trusted devices can connect. All your internet traffic will pass through this server, so securing it and keeping it updated is essential.  

## Steps to Deploy WireGuard VPN on AWS

### Create a AWS account
If you don’t already have an AWS account, sign up [here](https://aws.amazon.com/free/). AWS’s free tier offers 750 hours of EC2 instance usage every month for the first year.

Need help? This [YouTube tutorial](https://youtu.be/CjKhQoYeR4Q?si=hyiOuD5rOogGLJh0) will walk you through setting up an AWS account.


### Create an EC2 instance
1. Log in to the AWS Management Console.
2. Navigate to EC2 > Instances and click Launch Instance.
3. Choose the following options:  
    - AMI (Amazon Machine Image): Select Ubuntu Server.
    - Instance Type: Choose t2.micro for free-tier eligibility.
    - Region: Select Mumbai (or your preferred location).
4. Launch the instance and make note of the public IP address.

### Configure Firewall
AWS uses Security Groups to manage network traffic. Configure your instance’s security group to allow:

- UDP Port 51820 (for WireGuard traffic).
- SSH Port 22 (for initial setup and management).


#### Steps:
1. Go to EC2 > Security Groups.
2. Edit the security group associated with your instance.
3. Add the above rules.
4. After configuration, I recommend disabling SSH access for better security.

### DNS Configuration
If you have a domain name, you can point it to your AWS instance for easier access. If not, [DuckDNS](https://www.duckdns.org/) provides free dynamic DNS services.

To ensure your DNS updates automatically, create a simple cron job to update the DNS entries at regular intervals and on system reboot.

### Install PiVPN

[PiVPN](https://www.pivpn.io/) is an excellent tool for simplifying VPN server setup. While it was initially designed for Raspberry Pi, it works seamlessly on an EC2 instance.

To install PiVPN:

1. SSH into your EC2 instance:
```bash
ssh -i your-key.pem ubuntu@your-ec2-public-ip
```
2. Update the system:
```bash
sudo apt update && sudo apt upgrade -y
```
3. Install PiVPN:
```bash
curl -L https://install.pivpn.io | bash
```

4. Follow the prompts, selecting WireGuard as the VPN protocol.


### Add Devices
Once PiVPN is installed, you can add devices to your VPN:

1. Run the following command to create a new client profile:
```bash
pivpn add
```
2. PiVPN generates a .conf file or QR code for the device.
3. Use the WireGuard app (available on all major platforms) to import the profile.


## Enjoy Your VPN
With everything set up, you can now connect to your VPN and access Indian websites as if you were in India. Remember to turn off the VPN when you no longer need it, and keep your server updated for security.

By deploying a WireGuard VPN on AWS, you’ve gained a powerful tool for bypassing geo-restrictions and staying connected to services back home. Use it wisely!

