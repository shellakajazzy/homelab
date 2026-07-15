[![Entangled badge](https://img.shields.io/badge/entangled-Use%20the%20source!-%2300aeff)](https://entangled.github.io/)

# Homelab
The documentation, configuration files, and scripts for my homelab / private network.

## Architecture
This is the current plan of how my homelab / home network will be setup.

- Machines and boxes:
  - Dell PowerEdge T420 - Proxmox
    - OPNSense (Network Segmentation for Proxmox Containers)
    - Docker Swarm Manager (Deployment Server)
    - Pi-hole (DNS Server)
    - copyparty (File Share)
    - Forgejo (Git Forge)
    - Syncthing (File Sync)
    - Devbox(es)
      - One devbox will have GPU passthrough of a Radeon RX 480 for AI dev
    - Cybersecurity Labs
    - Running a RAID 5 array with 8 x 1TB hard drives
  - Raspberry Pi 3 - Raspberry Pi 3 OS
    - The services running on the RPi3 are ones that I want to ensure are always enabled
    - Pi-hole (DNS Server)
    - Syncthing (File Sync)
    - Logseq Sync Server

- Network rules:
  - VLAN Network Segments:
    - Only applies to the containers on the Proxmox server
    - Generally, these VLANs cannot access machines from each other or from the same network as the Proxmox host
      - If machines need to access each other, it is managed through the Tailnet and ACL rules
    - 42: Infrastructure
      - Basically just OPNSense, the Docker Swarm Manager, and the Pi-hole instance on the Proxmox server
      - Subnet: `42.42.42.0/24`
    - 67: Internal
      - Internal services that aren't really used for infrastructure
      - Subnet: `67.67.67.0/24`
    - 69: DMZ
      - Not necessarily exposed to the public internet, but still shared through Tailscale
      - Subnet: `69.69.69.0/24`
    - 100: Development
      - Self explanatory, in case I need multiple dev boxes.
      - Subnet: `100.100.100.0/24`
    - 200: Cybersecurity Labs
      - Might not be controlled via the OPNSense VM, but rather something else depending on how I choose to deploy labs.
      - Subnet: `200.200.200.0/24`
  - Tailnet ACL Rules:
    - | Source            | Can Access                             |
      | ----------------- | -------------------------------------- |
      | User Devices      | Any port on any Tailnet machine        |
      | Syncthing Devices | Any port on any other Syncthing device |
      | Swarm Worker      | Port 2377 on the Swarm Manager         |

- Deployment
  - All of the Swarm workers aside from the Raspberry Pi 3 are cloned from the same Proxmox container template, which is just an Ubuntu server container with Docker and Tailscale preinstalled.
  - The worker needs to be added to the swarm, and Tailscale serve / funnel needs to be setup manually.
  - The Raspberry Pi 3 is just running Raspberry Pi OS 3 and is joined to the swarm.
  - Just run the `docker swarm deploy -c` command to deploy / update the services controlled by the swarm.
  - Everything else is deployed manually.

## Swarm Manager
Every service in the swarm will be managed by this stack configuration file.

In order to join a worker to the swarm, get the join command with:
  `docker swarm join-token worker`

Ensure the worker is joined to the Tailnet and is tagged with the `swarm-worker` tag.

The swarm is deployed and updated by running:
  `docker stack deploy -c ./docker-stack.yml homelab`

[`docker-stack.yml`](./docker-stack.yml):
``` {.yaml file=./docker-stack.yml}
services:
  <<services>>

secrets:
  <<secrets>>
```

## Pi-hole
[Pi-hole](https://pi-hole.net/) will act as my DNS server.
Although I am mainly using it only for its blocklist capabilities, I might use it to play around with some DNS configurations.

`services`:
``` {.yaml #services}
pihole:
  image: pihole/pihole:latest
  ports:
    - target: 53
      published: 53
      mode: host
      protocol: tcp
    - target: 53
      published: 53
      mode: host
      protocol: udp
    - target: 443
      published: 443
      mode: host
      protocol: tcp
    - target: 80
      published: 80
      mode: host
      protocol: tcp
  environment:
    TZ: "America/Los_Angeles"
    WEBPASSWORD_FILE: /run/secrets/pihole_web_admin_pass
    FTLCONF_dns_listenMode: "ALL"
  volumes:
    - /srv/pihole:/etc/pihole
  deploy:
    replicas: 1
    placement:
      constraints:
        - node.hostname == pihole
```

I need to use swarm secrets in order to configure my Pi-hole web console password.

`secrets`:
``` {.yaml #secrets}
pihole_web_admin_pass:
  external: true
```

## copyparty
I am running [copyparty](https://github.com/9001/copyparty) as my fileshare.
I will most likely be sharing this instance through Tailscale so that I can provide some storage for friends.

`services`:
``` {.yaml #services}
copyparty:
  image: copyparty/ac:latest
  ports:
    - target: 3923
      published: 3923
      mode: host
      protocol: tcp
  command: ["-p", "3923", "-e2dsa", "-e2ts", "--usernames", "--ah-alg", "argon2"]
  volumes:
    - /srv/copyparty/copyparty.conf:/cfg/config.conf:ro
    - /srv/copyparty/db:/cfg/hists
    - /srv/copyparty/media:/media
  deploy:
    replicas: 1
    placement:
      constraints:
        - node.hostname == copyparty
```

In order to setup:
1. Edit `copyparty.conf`:
   ```
   [global]
     show-ah-salt
  
   [accounts]
     user: password
   ```
1. Deploy the stack:
   `docker stack deploy -c ./docker-stack.yml homelab`
1. Check the logs:
   `docker service logs homelab_copyparty`
1. Edit `copyparty.conf` to match the values in the logs:
   ```
   [global]
     ah-salt: <YOUR_AH_SALT>
    
   [accounts]
     user: +<USER_PASSWORD_HASH>
   ```
1. Repeat setups 2-4 every time a new user is added to find their hashed password

## Forgejo
I am using [Forgejo](https://forgejo.org/) as my main Git forge.
Like copyparty, I am likely going to share this with some friends.

`services`:
``` {.yaml #services}
forgejo:
  image: codeberg.org/forgejo/forgejo:15
  ports:
    - target: 3000
      published: 3000
      mode: host
      protocol: tcp
    - target: 22
      published: 22
      mode: host
      protocol: tcp
  environment:
    USER_PID: 1000
    USER_GID: 1000
  volumes:
    - /srv/forgejo:/data
    - /etc/localtime:/etc/localtime:ro
  deploy:
    replicas: 1
    placement:
      constraints:
        - node.hostname == forgejo
```
