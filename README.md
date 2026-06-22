# user-configs

This is the repository for my Custom Tipi user-config.

## Table of contents

- [user-configs](#user-configs)
  - [Table of contents](#table-of-contents)
  - [Overview](#overview)
  - [Prerequisites](#prerequisites)
  - [Installation](#installation)
  - [Getting started](#getting-started)
  - [Tips](#tips)
  - [Documentation](#documentation)
  - [Contribution](#contribution)
  - [Contact](#contact)
  - [License](#license)

## Overview

[Runtipi](https://runtipi.io) (also referred to just tipi) offers the opportunity to customize the configuration of
[served apps](https://runtipi.io/docs/guides/customize-app-config) and the used
[traefik reverse proxy](https://runtipi.io/docs/guides/customize-compose-and-traefik).

Runtipi is aimed at the use behind and via a VPN and or tunnel.
This repo contains a opinionated user-config that aims at having Runtipi:

- being used with a direct connection to the internet behind a NAT router
- having advanced configuration like SSO/MFA and rate limiting

## Credits

This repository is a fork of the original user-config project by Falk Heiland:

- [falkheiland/user-config](https://codeberg.org/falkheiland/user-config)

The structure, app patterns, and much of the configuration approach in this fork are based on that original work.

Placeholders used in this guide, app specific descriptions and in *.example files:

- example.com: actual domain name
- 192.168.1.0/24: actual private ip address range

## Prerequisites

- working [Runtipi](https://runtipi.io/docs/getting-started/installation) installation (tested with v4.1.0)
- installed [Authentik](https://github.com/goauthentik/authentik) app (tested with 2025.4.1)
- a domain managed via [Cloudflare](https://cloudflare.com) DNS with a wildcard entry (ie *.example.com) pointing at your  public IP address

It is also a good measure to follow up on OS hardening.
This highly depends on the used host OS and the hosting itself and is not part of this repo.

Since this comes up a lot, here is one thing related to security, when Runtipi is used directly on the internet:

- [ufw on ubuntu](https://github.com/chaifeng/ufw-docker?tab=readme-ov-file#problem):
  > But when Docker is installed, Docker bypass the UFW rules and the published ports can be accessed from outside
  - follow the steps from the link above
  - use your instance behind an additional firewall / nat router.
- Runtipi offers the ability to expose services to the local domain.
  If this local domain is spoofed and is set to the public IP, than those services are exposed!
  - use a traefik middleware that restricts the access to the effected traefik routers to local ip addresses

## Installation

- stop all apps in the runtipi GUI
- open the bash on your server and follow these steps:

```bash
# !!! this assumes runtipi was installed in your home directory.
# change dir to the runtipi dir
cd ~/runtipi
# stop runtipi
./runtipi-cli stop
# stop all containers (in case above command hangs )
docker stop $(docker ps -a -q)
# create dir structure for traefik app-data
mkdir -p ~/runtipi/app-data/traefik/shared
# display the traefik app-data dir structure
tree -L 5 -a app-data/traefik
app-data/traefik
└── shared
# clone the repo
# clone this fork into the user-config directory for your Runtipi instance
git clone https://github.com/rodrigomescua/runtupi-user-config.git user-config
# display the user-config dir structure (excerpt)
# tree -a -d -L 2 user-config/
user-config/
├── _archive
...
├── rodrigomescua
│   ├── arkanum
...
├── falkheiland-dev
├── .git
...
├── migrated
│   ├── 2fauth
...
│   └── wekan
└── traefik
    └── etc
...
# Find all env example files and rename them for use in runtipi
find ./user-config -type f -name "*.example" | while read file; do
    env_file="${file%.example}"
    mv "$file" "$env_file"
    echo "created env file $env_file"
done
# results (excerpt)
created env file ./user-config/rodrigomescua/authentik/app.env
...
created env file ./user-config/tipi.env
...

```

Open and edit each of the files from the result above in an editor of your choice. The `./user-config/tipi.env` contains the necessary env vars for the traefik reverse proxy - it is essential to edit this properly.

## Getting started

```bash
sudo ./runtipi-cli start
```

- open the traefik dashboard in your browser `http://<runtipi-IP>:8080/dashboard/#/`
  - there should be no errors shown for Routers, Services and Middlewares
  - if there are errors, fix them.
- open the runtipi GUI
- start the Authentik app
- open the Authentik GUI and make settings according to the README of each (used) app in the repo
- start each app after making above settings
- test the app

## Configuration Patterns

### Traefik Middleware Chains

The core of this setup is a set of Traefik middleware chains defined in `traefik/etc/dynamic/dynamic.yml`:

| Chain | Usage |
|---|---|
| `chain-Domain-auth@file` | Internet-facing apps — rate limit + security headers + Authentik ForwardAuth |
| `chain-Domain@file` | Internet-facing apps with own auth (e.g. Nextcloud) — no ForwardAuth |
| `chain-localDomain-auth@file` | Local network access + Authentik ForwardAuth |
| `chain-localDomain@file` | Local network only, no auth |

### Per-App Override Pattern

Each app gets a `rodrigomescua/<app>/docker-compose.yml` that overrides the Traefik labels Runtipi generates. The router name format is always `<app-id>-<username>` (e.g. `jellyfin-rodrigomescua`).

Minimal override for an internet-exposed app with Authentik ForwardAuth:

```yaml
services:
  <app-id>:
    labels:
      traefik.http.routers.<app-id>-rodrigomescua.middlewares: chain-Domain-auth@file
      traefik.http.routers.<app-id>-rodrigomescua.tls.certresolver: ''
```

For apps that need both internet access (with auth) and unauthenticated local network access, add a `-privip` router variant:

```yaml
services:
  <app-id>:
    labels:
      traefik.http.routers.<app-id>-rodrigomescua.middlewares: chain-Domain-auth@file
      traefik.http.routers.<app-id>-rodrigomescua.tls.certresolver: ''
      traefik.http.routers.<app-id>-rodrigomescua-privip.rule: Host(`${APP_DOMAIN}`) && ClientIP(`${PRIVATE_IPV4}`)
      traefik.http.routers.<app-id>-rodrigomescua-privip.middlewares: chain-Domain@file
```

## Tips

start tipi app without tipi

```bash
docker compose --env-file app-data/<app>/app.env --env-file user-config/<app>/app.env --project-name <app> -f apps/<app>/docker-compose.yml -f repos/<hash>/apps/docker-compose.common.yml --file user-config/<app>/docker-compose.yml up --detach --force-recreate --remove-orphans
```

update/recreate traefik container without affecting other containers (e.g. after updating the image version or changing env vars in `tipi.env`)

```bash
cd ~/runtipi && docker compose -f docker-compose.yml -f user-config/tipi-compose.yml up -d --no-deps --force-recreate runtipi-reverse-proxy
```

## Documentation

- Apps
  - rodrigomescua
    - [2FAuth](./rodrigomescua/2fauth/)
    - [Arkanum](./rodrigomescua/arkanum/)
    - [Authentik](./rodrigomescua/authentik/)
    - [Bookstack](./rodrigomescua/bookstack/)
    - [Cup](./rodrigomescua/cup/)
    - [CUPdate](./rodrigomescua/cupdate/)
    - [Cuppa](./rodrigomescua/cuppa/)
    - [Dawarich](./rodrigomescua/dawarich/)
    - [DDNS-Updater-CF](./rodrigomescua/ddns-updater-cf/)
    - [docker-db-backup](./rodrigomescua/docker-db-backup/)
    - [Dozzle](./rodrigomescua/dozzle/)
    - [Forgejo](./rodrigomescua/forgejo/)
    - [Freshrss-OIDC](./rodrigomescua/freshrss-oidc/)
    - [GHarmonize](./rodrigomescua/gharmonize/)
    - [Home Assistant](./rodrigomescua/homeassistant/)
    - [Immich](./rodrigomescua/immich/)
    - [IT-Tools](./rodrigomescua/it-tools/)
    - [Jellyfin](./rodrigomescua/jellyfin/)
    - [Joplin Server](./rodrigomescua/joplin/)
    - [Linkwarden](./rodrigomescua/linkwarden/)
    - [MeTube](./rodrigomescua/metube/)
    - [Moodist](./rodrigomescua/moodist/)
    - [Navidrome](./rodrigomescua/navidrome/)
    - [Networking Toolbox](./rodrigomescua/networking-toolbox/)
    - [Nextcloud-FPM](./rodrigomescua/nextcloud-fpm/)
    - [NocoDB](./rodrigomescua/nocodb/)
    - [Paperless-ngx](./rodrigomescua/paperless-ngx/)
    - [Pinchflat](./rodrigomescua/pinchflat/)
    - [PruneMate](./rodrigomescua/prunemate/)
    - [SearXNG](./rodrigomescua/searxng/)
    - [Sure](./rodrigomescua/sure/)
    - [Surmai](./rodrigomescua/surmai/)
    - [Uptime Kuma](./rodrigomescua/uptime-kuma/)
    - [Vaultwarden](./rodrigomescua/vaultwarden/)
    - [Wallos](./rodrigomescua/wallos/)
    - [Wekan](./rodrigomescua/wekan/)

***tbc***

## Contribution

Issues and PRs are welcome.

## Contact

use channel `user-configs` on the [Runtipi Discord](https://discord.gg/Bu9qEPnHsc).

## License

This project is licensed under the MIT License.
