<div align="center" style="white-space: nowrap;">
  <img src="https://github.com/4LifeStrategy/4LifeStrategy/blob/88ffe3009f1399de4502d4d5641c8f7a0fd56852/4LifeStrategy%20Logo%20Center.png" alt="4LifeStrategy Logo" width="100" style="display:inline-block; vertical-align:middle; margin-right:10px;">
  <h1 style="margin:0; vertical-align:middle;">Reverse Proxy Configuration on Synology NAS</h1>
</div>

## Description

**Reverse Proxy** is a server that sits in front of web servers and forwards client (e.g. web browser) requests to those web servers.

## Project Scope

**Objective**: Configured a reverse proxy using Nginx Proxy Manager with SSL certificates managed through Cloudflare. Enhanced security and optimized traffic management for web applications.

## Configuring Macvlan Network Interface

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
3. For **Project Name** put **nginx-proxy-manager**
4. For **Path** set to the shared folder we created **/docker/nginx-proxy-manager**
5. For **Source** select **Create docker-compose.yml**
6. Copy and modify code bellow to fit your macvlan configuration
```
version: "3"
services:
  npm:
    container_name: nginx-proxy-manager
    image: 'jc21/nginx-proxy-manager:latest'
    restart: unless-stopped
    ports:
      - '80:80' # Public HTTP Port
      - '443:443' # Public HTTPS Port
      - '81:81' # Admin Web Port
    volumes:
      - /volume1/docker/nginx-proxy-manager/data:/data
      - /volume1/docker/nginx-proxy-manager/letsencrypt:/etc/letsencrypt
    environment:
      DISABLE_IPV6: 'true'
    networks:
     npm_zbridge:
       ipv4_address: 192.168.100.10
       priority: 900
     npm_network:
       ipv4_address: 192.168.251.3 # Modify
       priority: 1000
networks:
    npm_zbridge:
      name: npm_zbridge
      driver: bridge
      ipam:
        config:
          - subnet: 192.168.100.0/24
            gateway: 192.168.100.1
            ip_range: 192.168.100.0/24
    npm_network:
      name: nginx_network
      driver: macvlan
      driver_opts:
        parent: ovs_eth0
      ipam:
        config:
          - subnet: 192.168.251.0/24 # Modify
            ip_range: 192.168.251.198/32 # Modify
            gateway: 192.168.251.1 # Modify
```
7. Click **Next**
8. Click **Next**
9. Click **Done**

## Configure SSL Certificate

Open a web browser and enter the ip address configured to access Nginx Proxy Manager instance. Enter the default email: admin@example.com and default password: changeme. You will be prompted to create an admin account with your email and password. Next we are going to configure SSL certificates. For this setup we are using Cloudflare for our domain's dns resolver and will need your API key.

1. Click **SSL Certificates**
2. Click **Add SSL Certificate**
3. Select **Let's Encrypt**
4. For **Domain Name** enter your domain with(*) in the beginning ***4lifestrategy.com**
5. Toggle **Use a DNS Challenge**
6. For **DNS Provider** select **Cloudflare**
7. Remove the API key placeholder and pass your **API key**<br /><img src="https://github.com/4LifeStrategy/Reverse-Proxy-Configuration/blob/84ea754e989a4835d1ad78e6f7e7b33014aef13c/NPM%20DNS%20Challenge.png" width="500">
8. For **Propagation Seconds** put **20**
9. Click **Save**

## Configure Proxy Hosts

1. Click **Hosts** 
2. Click **Add Proxy Host**
3. For **Domain Name** enter **nginx.4lifestrategy.com**
4. For **Scheme** select **http**
5. For **Forward Hostname / IP** enter **Nginx Proxy Manager's IP address**
6. For **Forward Port** enter **81**
7. Enable **Cache Assets**
8. Enable **Block Common Exploits**
9. Enable **Websockets Support**<br /><img src="https://github.com/4LifeStrategy/Reverse-Proxy-Configuration/blob/e76798fbc4fb2cf2d836515e028ca0efca962654/NMP%20Poxy%20Host.png" width="500">
10. Select **SSL** tab
11. For **SSL Certificate** select the **Certificate we created**
12. Enable **Force SSL**
13. Enable **HSTS Enabled**
14. Enable **HTTP/2 Support**
15. Enable **HSTS Subdomains**<br /><img src="https://github.com/4LifeStrategy/Reverse-Proxy-Configuration/blob/76d677c56b0b49715a988d4b3869a0b2fe13a38d/NPM%20Proxy%20Host%20SSL.png" width="500">
16. Click **Save**

Next we are going to create a Proxy Host for Synology DSM, and this will apply to any other service running off the Synology NAS that is assessed off a specified port. 

1. Click **Hosts** 
2. Click **Add Proxy Host**
3. For **Domain Name** enter **dsm.4lifestrategy.com** 
4. For **Scheme** select **https**
5. For **Forward Hostname / IP** enter **192.168.100.1**<br />*This ip is what we configured to bridge services between the macvlan and synology actual ip address.*
6. For **Forward Port** use **5051**
7. Enable **Cache Assets**
8. Enable **Block Common Exploits**
9. Enable **Websockets Support**<br /><img src="https://github.com/4LifeStrategy/Reverse-Proxy-Configuration/blob/3a977b7becf6013d54b99f090635844e5556bc8f/NPM%20dsm.png" width="500">
10. Select **SSL** tab
11. For **SSL Certificate** select the **Certificate we created**
12. Enable **Force SSL**
13. Enable **HSTS Enabled**
14. Enable **HTTP/2 Support**
15. Enable **HSTS Subdomains**
16. Click **Save**
