# README

## Secure Remote Access to Home Network Services

On the home server we need to have a secure connection to the web cameras. We want to limit who can access the cameras, and we don't want to run a VPN. And while we're at it, we might as well run the Pi-hole DNS blackhole on it, and point the IoT devices to use the Pi-hole.

Cloudflare tunnels work well, and I've used them for more than a year. Cloudflare now calls this service, [Cloudflare Zero Trust](https://developers.cloudflare.com/cloudflare-one/applications/configure-apps/#:~:text=Cloudflare%20Docs-,Cloudflare%20Zero%20Trust,-Cloudflare%20Zero%20Trust), because hey, everyone wants zero trust now days.

I had to update my edge Raspberry Pi OS to Ubuntu 20.10, and decided to add [Pi-hole](https://docs.pi-hole.net/) to it as well. And since I'm upgrading, I wanted to up my game and finally learn to use [Docker](https://docker.com). So, I had to install [cloudflared](https://hub.docker.com/r/msnelling/cloudflared), [nginx](https://hub.docker.com/_/nginx), and [pihole](https://hub.docker.com/r/pihole/pihole) containers on the Raspberry Pi 3 B from 2015. (Aren't these things amazing?)

### Network Diagram for Cloudflare Tunnels

The diagram shows the connectivity between the system compoents, both internal and external, physical and virtual.

When the cloudflared container is created, it creates a bridge network named cloudflared. The nginx and pihole containters are added to cloudflared network so they can be available over the cloudflare tunnel created by the cloudflared container.

The camera network will connect to the nginx container, restricting access to the camera webservers to the cloudflared tunnel. These are what Cloudflare calls [Self-host applications](https://developers.cloudflare.com/cloudflare-one/applications/configure-apps/self-hosted-apps/) and authentication can be controlled by your IdP such as Okta or Google for example. The diagram below shows the connectivity between the system components.

![Cloudflare diagram of how traffic flows between users, Cloudflare, Identity Providers, and self-hosted applications](https://developers.cloudflare.com/cloudflare-one/static/documentation/applications/network-diagram.png)

### Diagram of Home Network

The diagram below shows the connections on the server and to internal and external sites.

![diagram of the network connections on the server and to internal and external sites](pihole-server-diagram.png)

## Pihole Web Interface

![screenshot of web interface for pihole](pihole-server.png)

## Repo Contents

This repo has the code to create services for our home server running:

* cloudflared - to create a private tunnel with a proxy managed by Cloudflare
* nginx - a reverse proxy for the camera webpages
* pihole - a DNS sinkhole and ad blocker
* cloudflared-host - tunnel to the host for SSH

## How to Install

---

### Prerequisites

1. Ubuntu 20.10
1. Docker installed and running

---

### Manage Docker as a non-root user

1. Create the `docker` group:
`sudo groupadd docker`
1. Add your user to the `docker` group:
`sudo usermod -aG docker $USER`
1. Logout and log back in
1. Verify that you can run `docker` without `sudo`:
`docker run hello-world`

See [Instructions from Docker](https://docs.docker.com/engine/install/linux-postinstall/) if there's any trouble.

### Build and Start Docker Containers

Clone this repo to host in your home directory
`gh repo clone beckitrue/pihole`

#### cloudflared

1. Retrieve cloudflared token from password manager and edit the `docker-compose.yaml` file
1. `vi ~/cloudflared/docker-compose.yaml`
1. Build and start cloudflared docker container `docker compose -f ~/cloudflared/docker-compose.yaml up -d`
1. Go to [Tunnels dashboard](https://one.dash.cloudflare.com/699b49d3fee8e9138a49442ea0119cb6/access/tunnels) and verify that tunnel is healthy.

#### nginx

1. Build and start nginx docker container `docker compose -f ~/nginx/docker-compose.yaml up -d`
1. Go to [nginx webpage](https://cameras.beckitrue.com/) and verify it's working.

#### pihole

1. Retrieve pihole WEBADMIN password from password manager and edit the `docker-compose.yaml` file
1. `vi ~/pihole/docker-compose.yaml`
1. Build and start pihole docker container `docker compose -f ~/pihole/docker-compose.yaml up -d`
1. Go to [pihole admin page](https://pihole.beckitrue.com/admin/index.php) and verify it's working
1. Follow the instructions for [Installing on Ubuntu](https://github.com/pi-hole/docker-pi-hole#installing-on-ubuntu-or-fedora) on the pi-hole GitHub site. This is to make the pi-hole the DNS server running on the Raspberry Pi.

#### cloudflared-host

1. Retrieve cloudflared token from password manager and edit the `docker-compose.yaml` file
1. `vi ~/cloudflared-host/docker-compose.yaml`
1. Build and start cloudflared docker container `docker compose -f ~/cloudflared-host/docker-compose.yaml up -d`
1. Go to [Tunnels dashboard](https://one.dash.cloudflare.com/699b49d3fee8e9138a49442ea0119cb6/access/tunnels) and verify that tunnel is healthy.

### Rebuild Containers

1. Pull latest image: `docker pull pihole/pihole`
1. `docker-compose up --build --remove-orphans --force-recreate -d`

## ToDo

* Add nginx config for web page to cameras - dependent on getting new home network set up
* Deploy with Terraform
