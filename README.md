# NordVPN Wireguard Client Config (w/ Docker Nordlynx Alternative) HOW-TO

## Table of Contents

- [Description](#description)
- [Getting Started](#getting-started)
  - [Prerequisites](#prerequisites)
  - [Set-Up](#setup)
  - [Create config file](#create-config)
- [Usage](#usage)
  - [Docker-compose example](#docker-compose-example)
  - [Raspberry Pi notes](#running-on-a-raspberry-pi)
  - [Connecting other containers](#routing-other-containers-through-your-nordlynx-container)
- [Acknowledgements](#acknowledgements)

## Description

Detailed walkthrough of how to create your own NordVPN wireguard configuration file for use with wireguard client software (including Docker).

## Getting Started

So you are a NordVPN customer who, for whatever reason, does not want to use their NordVPN local client to access their Wireguard servers.

Maybe you want to set up a Docker client to route your other Docker containers?  Maybe you just dont want to run nordvpn's binaries at all time on your host machine? 

Currently, NordVPN refuses to provide a Wireguard configuration file that you can use to access their wireguard servers with your own Wireguard client application on various devices.

Here I'll attempt to walk you through the steps to create your own NordVPN wireguard configuration file, which you can drop in to various instances of Wireguard (later in this example, we'll use Docker).

### Prerequisites

Linux OS (ubuntu in this example)

NordVPN Account

### Setup

1. Install following packages on your machine:

    wireguard: ```sudo apt install wireguard```

    curl: ```sudo apt install curl```

    jq: ```sudo apt install jq```

    nettools: ```sudo apt install net-tools```

    nordvpn: ```sh <(curl -sSf https://downloads.nordcdn.com/apps/linux/install.sh)```

2. Log in to your nordvpn app via this command:
```sudo nordvpn login```

3. Change connection protocol to nordlynx:
```sudo nordvpn set technology nordlynx```

4. Connect to your preferred server
```sudo nordvpn c nl #to connect Nederland as an example```

5. Now run command below and write down the ip somewhere, you'll need it later. (it will be your wireguard interface ip)
```ifconfig nordlynx```

<img src="https://forum.openwrt.org/uploads/default/original/3X/d/2/d203de32a20b063ce11f03407538c96abb4bc991.png">

6. Use the following command to get your private key:

```sudo wg show nordlynx private-key```

output should be something like this:

<img src="https://forum.openwrt.org/uploads/default/original/3X/3/8/38407e183d31e9b52617f31d70424047156d3a1e.png">

**Never share your private key with anyone!**

7. Now get your fastest Nordvpn server:

```curl -s "https://api.nordvpn.com/v1/servers/recommendations?&filters\[servers_technologies\]\[identifier\]=wireguard_udp&limit=1"|jq -r '.[]|.hostname, .station, (.locations|.[]|.country|.city.name), (.locations|.[]|.country|.name), (.technologies|.[].metadata|.[].value), .load'```

<img src="https://forum.openwrt.org/uploads/default/original/3X/1/c/1c3402ae4864e6499a67b9e1cc4fb6f4a97b79f4.png">

output should be like above picture which includes
nl826.nordvpn.com 56 #your endpoint host
178.239.173.207 #endpoint host ip
Amsterdam #city
Netherlands #country
5PXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXF30= #public key which is different for each server
9 #server load at the time

**Congrats, you now have all the information needed to create your own wireguard config file!**

## Create config

1. Create a file named wg0.conf with your favourite text editor (I like nano):
```nano wg0.conf```
2. Add the following text, replacing "xxx" where indicated: 
```
[Interface]
Address = 10.5.0.2/32 #interface found in step 5, replace if different for you
PrivateKey = xxx #private key found in step 6
DNS = 9.9.9.9 #dns provider of your choice, here i am using quad9

[Peer]
PublicKey = xxx #public key of chosen server, found in step 7
#PresharedKey = [Pre-shared key, same for server and client] # pre-shared key doesnt seem to be needed for nordvpn servers
Endpoint = xxx:51820 #IP address of fastest server found in step 7, using port 51820
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 21
```
3. Save your wg0.conf (ctrl+o)
4. Drop this file into the appropriate directory for your wireguard client setup (continue following along if you want to use Docker client)

## Usage

### Docker-compose (example)

Currently, one of the popular solutions for connecting to NordVPN over wireguard in a Docker container is to use <a href="https://github.com/bubuntux/nordlynx">@bubuntux's nordlynx container</a>.  Unfortunately for some (myself included) this container has some strange performance issues which result in unusually slow speeds for no apparent reason.  Props to bubuntux for making the effort to release a comprehensive and user friendly solution, however it seems at this moment that a fix to the performance issues is not imminent.

The alternative I'm proposing is to simply drop our own wireguard configuration file (created above) into a vanilla Linuxserver.IO Wireguard docker container.  Here's a docker-compose example to get you started:

1. Use your favourite text editor to create a docker-compose.yml file:
```nano docker-compose.yml```
2. Copy the following:
```version: "3.4"
networks:
  nord-network:
    driver: bridge
    ipam:
      driver: default
      config:
        - subnet: "172.30.172.0/24"
    enable_ipv6: false
services:
  nordlynx:
    image: lscr.io/linuxserver/wireguard:latest
    container_name: nordlynx
    cap_add:
      - NET_ADMIN
      - SYS_MODULE
    environment:
      - NET_LOCAL=192.168.1.0/24 #your host machine subnet
      - PUID=1000
      - PGID=1000
      - TZ=US/Eastern #replace with your TZ
    ports:
      - 51820:51820/udp #wireguard UDP port
    sysctls:
      - net.ipv6.conf.all.disable_ipv6=1
      - net.ipv4.conf.all.src_valid_mark=1
    networks:
      - nord-network
    restart: unless-stopped
    volumes:
      - ./config:/config:ro
      - /lib/modules:/lib/modules
```

3. Save your docker-compose.yml file (ctrl+o)
4. Create a config directory in the same location as your docker-compose.yml file:
```mkdir config```
5. Copy the wg0.conf file created earlier into this config directory.

This will ensure wireguard runs in client mode and connects to the server specified in the config file.

6. Check your Wireguard instance is up and running properly:
```docker logs nordlynx```

This should output something along these lines:
```s6-rc: info: service legacy-cont-init successfully started
s6-rc: info: service init-mods: starting
s6-rc: info: service init-mods successfully started
s6-rc: info: service init-mods-package-install: starting
s6-rc: info: service init-mods-package-install successfully started
s6-rc: info: service init-mods-end: starting
s6-rc: info: service init-mods-end successfully started
s6-rc: info: service init-services: starting
s6-rc: info: service init-services successfully started
s6-rc: info: service legacy-services: starting
services-up: info: copying legacy longrun wireguard (no readiness notification)
s6-rc: info: service legacy-services successfully started
s6-rc: info: service 99-ci-service-check: starting
[ls.io-init] done.
s6-rc: info: service 99-ci-service-check successfully started
Warning: `/config/wg0.conf' is world accessible
[#] ip link add wg0 type wireguard
[#] wg setconf wg0 /dev/fd/63
[#] ip -4 address add 10.5.0.2/32 dev wg0
[#] ip link set mtu 1420 up dev wg0
[#] resolvconf -a wg0 -m 0 -x
[#] wg set wg0 fwmark 51820
[#] ip -4 route add 0.0.0.0/0 dev wg0 table 51820
[#] ip -4 rule add not fwmark 51820 table 51820
[#] ip -4 rule add table main suppress_prefixlength 0
[#] sysctl -q net.ipv4.conf.all.src_valid_mark=1
sysctl: setting key "net.ipv4.conf.all.src_valid_mark", ignoring: Read-only file system
```
If you dont see any major errors, you're likely up and running.

### Running on a raspberry pi

If you are running your docker containers on a raspberry pi like me, you may run into issues with your wireguard containers not working.  Try upgrading to 64bit Raspbian and installing raspberry pi kernel headers on your host system:

```sudo apt install raspberrypi-kernel-headers```

### Routing other containers through your Nordlynx container

If you want to be able to route internet access for your other docker containers through your NordVPN wireguard instance (while still being able to maintain local network access to these containers), you need to make some further adjustments.

1. Open your wg0.conf file with your text editor:
```nano wg0.conf```

2. Add the following lines to your configuration file, under the [Interface] heading:
 ```
PostUp = DROUTE=$(ip route | grep default | awk '{print $3}'); HOMENET=192.168.0.0/16; HOMENET2=10.0.0.0/8; HOMENET3=172.16.0.0/12; ip route add $HOMENET3 via $DROUTE;ip route add $HOMENET2 via $DROUTE; ip route add $HOMENET via $DROUTE;iptables -I OUTPUT -d $HOMENET -j ACCEPT;iptables -A OUTPUT -d $HOMENET2 -j ACCEPT; iptables -A OUTPUT -d $HOMENET3 -j ACCEPT;  iptables -A OUTPUT ! -o %i -m mark ! --mark $(wg show %i fwmark) -m addrtype ! --dst-type LOCAL -j REJECT
PreDown = HOMENET=192.168.0.0/16; HOMENET2=10.0.0.0/8; HOMENET3=172.16.0.0/12; ip route del $HOMENET3 via $DROUTE;ip route del $HOMENET2 via $DROUTE; ip route del $HOMENET via $DROUTE; iptables -D OUTPUT ! -o %i -m mark ! --mark $(wg show %i fwmark) -m addrtype ! --dst-type LOCAL -j REJECT; iptables -D OUTPUT -d $HOMENET -j ACCEPT; iptables -D OUTPUT -d $HOMENET2 -j ACCEPT; iptables -D OUTPUT -d $HOMENET3 -j ACCEPT
```

These were lifted from LSIO's wireguard instructions and are labelled as "unsupported, use at your own risk".  You may need to alter the subnets as needed, however this appears to work as-is in my configuration.

3. Save your config file (ctrl+o).  
4. Now edit your docker-compose file as needed and add other services to the stack.  Any ports you want to expose for other services (say, for example, port 8080 for qbittorrent web UI) will need to be published in your "nordlynx" container.  In order to make other services use the nordlynx container, add this within the other services' section:
```
    cap_add:
      - NET_ADMIN
    network_mode: service:nordlynx
    depends_on:
      - nordlynx
```

6. Recreate your wireguard container:
```docker-compose up -d --force-recreate```

Now your stack should launch nordlynx first, followed by anything else that depend on it.  This configuration will have all your containers in the stack use the same local IP address and the same network.  They can now talk to each other using "localhost:port" (or 127.0.0.1:port) and you can reach the containers from other machines on your local network using (example) http://host-local-ip:port 

Soon i'm going to upload an example of my current docker stack: nordvpn-qbittorrent-sonarr-radarr-prowlarr-bazaar, forked from the pjortiz stack linked below.  

## Acknowledgements

MAJOR shout out to "Armin" on OpenWRT forums who <a href="https://forum.openwrt.org/t/instruction-config-nordvpn-wireguard-nordlynx-on-openwrt/89976">posted the initial guide</a> which got me started on this process.  I basically lifted the first several steps of the setup from his original post word-for-word because everything was well laid out and well described, along with his original images and simply made some formatting changes before further adapting it.  Hope he doesnt mind!

Also thanks to bubuntux whose original <a href="https://github.com/bubuntux/nordlynx">Nordlynx container</a> gave me the idea that it would be possible to use NordVPN with docker containers at all!

Thanks to Linuxserver.IO team for making great docker container images including the <a href="https://github.com/linuxserver/docker-wireguard">wireguard image</a> used here.

And thanks to <a href="https://github.com/pjortiz/docker-arrs-with-nordvpn">pjortiz docker-arrs-with-nordvpn</a> docker stack which got me up and running with my working docker stack config.  I simply altered his stack and dropped in LSIO's wireguard instead of bubuntux's Nordlynx container and made some slight adjustments.
