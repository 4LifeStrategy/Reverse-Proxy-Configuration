<div align="center" style="white-space: nowrap;">
  <img src="https://github.com/4LifeStrategy/4LifeStrategy/blob/88ffe3009f1399de4502d4d5641c8f7a0fd56852/4LifeStrategy%20Logo%20Center.png" alt="4LifeStrategy Logo" width="100" style="display:inline-block; vertical-align:middle; margin-right:10px;">
  <h1 style="margin:0; vertical-align:middle;">Reverse Proxy Configuration on Synology NAS</h1>
</div>

## Description

**Reverse Proxy** is a server that sits in front of web servers and forwards client (e.g. web browser) requests to those web servers.

## Project Scope

**Objective**: Configured a reverse proxy using Nginx Proxy Manager with SSL certificates managed through Cloudflare. Enhanced security and optimized traffic management for web applications.

## Configuring a Macvlan Network Interface

As mentioned above, we’re configuring a macvlan network interface so that our Nginx Proxy Manager instance will have an entirely separate IP address and ports.

**Enable SSH**
1. Open **Control Panel**
2. Click **Terminal & SNMP**
3. Enable **Enable SSH service**
4. Click **Apply**

**SSH into NAS and Configure Macvlan**

You will need a terminal to ssh into the Snology NAS. In this demonstration we are using a linux terminal.


1. Open **Terminal** on you machine
2. Enter the command
 
        ssh [username]@[hostname or IPaddress]

3. Run the command below and note down the network interface name that has your Synology NAS’s IP address (in this example, mine is **eth0**)

        ifconfig

4. Run the command below while substituting the correct subnet*(we configured 192.168.251..0/24 when setting up pfSense because the gateway would be 192.168.215.1)*. You also need to pick an IP address that you’d like to use that’s not currently in use. I will be using *192.168.215.3*.

        sudo docker network create -d macvlan -o parent=eth0 --subnet=192.168.215.0/24 --gateway=192.168.215.1 --ip-range=192.168.215.198/32 nginx_network

*For security I would go back to **Terminal & SNMP** and disable SSH*

## Installation

We are going to deploy Nginx Proxy manager using synology Container Manager which runs off Docker.

1. Open **Package Manager**
2. Search **Container**
3. Select **Container Manager**
4. Click **Install**

It would have create a docker shared folder. We are going to create a sub directory for our nginx proxy manager project.

1. Open **File Station**
2. Click **Create**
3. Select **Create folder**
4. Name it **nginx-proxy-manager**
5. Click **OK**
6. Enter the **nginx-proxy-manager** folder we created
7. Create 2 folders, one named **data** and other named **letsencrypt**<br /><img src="https://github.com/4LifeStrategy/Reverse-Proxy-Configuration/blob/922162a849799e4f22d16810f620a3751ce3243a/Nginx%20Folders.png" width="500">

Next are going to obtain the Nginx Proxy Manager's docker image.

1. Open **Container Manager**
2. Select **Registry**
3. Search **nginx-proxy-manager**
4. Select **jc21/nginx-proxy-manager**
5. Click **Download**
6. Select **latest**
7. Click **Apply**

We will be utilizing a Docker Compose file to create the entire Nginx Proxy Manager container, which will contain all of its configurations.

1. With **Container Manager** select **Project**
2. Click **Create**
