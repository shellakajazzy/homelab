[![Entangled badge](https://img.shields.io/badge/entangled-Use%20the%20source!-%2300aeff)](https://entangled.github.io/)

# Homelab
The documentation, configuration files, and scripts for my homelab / private network.

Everything is pretty much in a single Docker compose file.

[`docker-compose.yaml`](./docker-compose.yaml):
```{.yaml file=docker-compose.yaml}
services:
    <<compose-services>>
```

## DockTail

`compose-services`:
``` {.yaml #compose-services}
docktail:
    image: ghcr.io/marvinvr/docktail:latest
    container_name: docktail
    restart: unless-stopped
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock:ro"
      - "/var/run/tailscale:/var/run/tailscale"
    environment:
      - "TAILSCALE_OAUTH_CLIENT_ID=${TAILSCALE_OAUTH_CLIENT_ID}"
      - "TAILSCALE_OAUTH_CLIENT_SECRET=${TAILSCALE_OAUTH_CLIENT_SECRET}"
```

## copyparty

`compose-services`:
``` {.yaml #compose-services}
copyparty:
    image: copyparty/ac:latest
    container_name: copyparty
    restart: unless-stopped
    user: "1000:1000"
    ports:
        - "3923:3923"
    labels:
        - "docktail.service.enable=true"
        - "docktail.service.name=copyparty${SERVICE_SUFFIX}"
        - "docktail.service.port=3923"
        - "docktail.service.service-port=443"
    volumes:
        - "${COPYPARTY_CONFIG:-./copyparty-dev.conf}:/cfg/copyparty.conf:z"
        - "${COPYPARTY_SHARE}:/share:z"
```

Here is a sample copyparty share config

[`copyparty-dev.conf`](./copyparty-dev.conf):
``` {.conf file=copyparty-dev.conf}
[global]
  e2dsa
  e2ts
  ansi
  xff-src: lan

[accounts]
  dev: dev

[/]
  /share
  accs:
    rwmda: dev
  flags:
    e2ds
```
