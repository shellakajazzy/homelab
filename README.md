[![Entangled badge](https://img.shields.io/badge/entangled-Use%20the%20source!-%2300aeff)](https://entangled.github.io/)

# Homelab
The documentation, configuration files, and scripts for my homelab / private network

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