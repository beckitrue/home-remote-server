# README

## Project Requirements and Goals

* Secure, remote access to cameras
* Secure access with IdP (Okta, Google, etc.)
* Nginx reverse proxy to access cameras
* Secure, remote access to host for management (I use this even when I'm on network)
* Applications run in containers
* Run on older Raspberry Pi (armv7l)
* Bonus: Pi-hole
* Learn how to run multiple containers and networks on a single host
* Learn to use .env file to configure secrets in docker compose using 1Password CLI
* Learn to use service account with 1Password CLI

## Secure Remote Access to Home Network Services

On the home server we need to have a secure connection to the web cameras. We want to limit who can access the cameras, and we don't want to run a VPN. And while we're at it, we might as well run the Pi-hole DNS blackhole on it and point the IoT devices to use the Pi-hole. Finally, I'd like to have remote access to the server, so I added a 2nd Cloudflare tunnel so I can SSH to the host.

Cloudflare tunnels work well, and I've used them for more than a year. Cloudflare now calls this service, [Cloudflare Zero Trust](https://developers.cloudflare.com/cloudflare-one/applications/configure-apps/#:~:text=Cloudflare%20Docs-,Cloudflare%20Zero%20Trust,-Cloudflare%20Zero%20Trust), because hey, everyone wants zero trust now days.

I had to update my edge Raspberry Pi OS to Ubuntu 22.10, and decided to add [Pi-hole](https://docs.pi-hole.net/) to it as well. And since I'm upgrading, I wanted to up my game and finally learn to use [Docker](https://docker.com).

I had to install [cloudflared](https://hub.docker.com/r/msnelling/cloudflared), [nginx](https://hub.docker.com/_/nginx), and [pihole](https://hub.docker.com/r/pihole/pihole) containers on the Raspberry Pi 3 B from 2015. (Aren't these things amazing?).

I'm using a Docker image `msnelling/cloudflared` because I'm running this on an older Pi, but if you're using a newer one, you can use the official Cloudflare image `cloudflare/cloudflared:latest`. You can configure this as environment variable in the `.env` file.

### Network Diagram for Cloudflare Tunnels

The diagram shows the connectivity between the system components, both internal and external, physical and virtual.

When the cloudflared container is created, it creates a bridge network named `cloudflared`. The nginx and pihole containters are added to cloudflared network so they can be available over the Cloudflare tunnel created by the cloudflared container.

The camera network connects to the nginx container, restricting access to the camera webservers to the cloudflared tunnel. These are what Cloudflare calls [Self-hosted applications](https://developers.cloudflare.com/cloudflare-one/applications/configure-apps/self-hosted-apps/) and authentication can be controlled by your IdP such as Okta or Google for example. The diagram below shows the connectivity between the system components.

![Cloudflare diagram of how traffic flows between users, Cloudflare, Identity Providers, and self-hosted applications](https://developers.cloudflare.com/assets/network-diagram_hu35c98d3bbf0ecf738b5b543af7009e44_79161_2296x1101_resize_q75_box_3-fe1feb83.png)

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
* .env file with secrets and image name for cloudflared

## How to Install

I recommend you break this down into pieces if you're using Cloudflare Tunnels for the first time. Start with the `cloudflared-host` container that connects you to the Raspberry Pi host and see if you can SSH to it, then move on to the other components. You could also hard code your secrets to make things easier to test, and then go back and change the code to use 1Password.

---

### Prerequisites

1. [Ubuntu 22.10](https://ubuntu.com/download/raspberry-pi)
1. [Docker installed](https://docs.docker.com/engine/install/ubuntu/) and running
1. [docker-compose installed](https://docs.docker.com/compose/install/linux/)
1. [Cloudflare account and domain](https://dash.cloudflare.com/sign-up)
1. [Configure your application(s)](https://developers.cloudflare.com/cloudflare-one/applications/configure-apps/) in Cloudflare
1. [1Password CLI installed](https://1password.com/downloads/command-line/)

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
`gh repo clone beckitrue/home-remote-server`

#### Configure .env Values

We're saving our secrets in an .env file and referencing them with 1Password. This ensures that the secrets don't get saved to GitHub. Instructions for referencing 1Password values can be found [here](https://developer.1password.com/docs/cli/get-started#usage). It's a good idea to run `op read op://<your credential>` to verify that your credential reference is correct.

1. Create a 1Password [service account](https://developer.1password.com/docs/service-accounts/get-started/)
1. Set your op service account environment variable `export OP_SERVICE_ACCOUNT_TOKEN=<token>`
1. Save your cloudflared tokens and your PiHole web password in 1Password
1. Change the values in the .env file to the correct values for your cloudflared tokens and PiHole password using your 1Password credential references
1. Set the cloudflared image you want to use. The official image is `cloudflare/cloudflared:latest`. The image I'm using works on the older arm7l Pi architecture.

#### cloudflared-host

1. Build and start cloudflared docker container from the `home-remote-server` directory run `op run --env-file=".env" -- docker compose -f cloudflared-host/docker-compose.yaml up -d`
1. Go to [Tunnels dashboard](https://one.dash.cloudflare.com) and verify that tunnel is healthy.
1. SSH to the host using the Cloudflare tunnel `https://your_app.yourdomain`

#### cloudflared

1. Build and start cloudflared docker containerfrom the `home-remote-server` directory run `op run --env-file=".env" -- docker compose -f cloudflared/docker-compose.yaml up -d`
1. Go to [Tunnels dashboard](https://one.dash.cloudflare.com) and verify that tunnel is healthy.

#### nginx

We're [manging the Nginx configuration and content files by mounting to a local directory](https://docs.nginx.com/nginx/admin-guide/installing-nginx/installing-nginx-docker/#maintaining-content-and-configuration-files-on-the-docker-host) on the Pi. Any changes in those directories will show up in the container. You'll restart your containter to pull the config and / or new content.

1. `sudo mv ~/home-remote-server/nginx/default.conf /var/nginx/conf/default.conf` This is where you'll change your nginx config file(s)
1. `sudo mv ~/home-remote-server/nginx/index.html /var/www/index.html` This is where you'll change your content file(s)
1. Build and start nginx docker container `docker compose -f ~/home-remote-server/nginx/docker-compose.yaml up -d`
1. Go to [nginx webpage](https://cameras.beckitrue.com/) and verify it's working. (use your URL)

#### pihole

1. Build and start pihole docker container from the `home-remote-server` directory run `op run --env-file=".env" -- docker compose -f pihole/docker-compose.yaml up -d`
1. Go to [pihole admin page](https://pihole.beckitrue.com/admin/index.php) (use your own URL) and verify it's working
1. Follow the instructions for [Installing on Ubuntu](https://github.com/pi-hole/docker-pi-hole#installing-on-ubuntu-or-fedora) on the pi-hole GitHub site. This is to make the pi-hole the DNS server running on the Raspberry Pi.
1. Follow the [Post-Install](https://docs.pi-hole.net/main/post-install/) instructions to complete the configuration **Make sure you have connectivity and the pi-hole is resolving DNS before making these changes, or you may not have DNS available**
1. [Enable the default blocklist](https://discourse.pi-hole.net/t/restoring-default-pi-hole-adlist-s/32323) to get started. Follow the instructions for Pi-Hole V5.

### Rebuild Containers

1. Pull latest image: `docker pull pihole/pihole`
1. `cd ~/home-remote-server/pihole`
1. `op run --env-file="../.env" -- docker-compose up --build --remove-orphans --force-recreate -d`

## Troubleshooting

Running these services in Docker containers was meant to be a learning experience for me, and it was. Here are some troubleshooting steps I had to take to troubleshoot DNS, network connectivity, and re-creating containers to use configuration changes.

### Assumptions

* Assumes that your Cloudflare tunnel is working and you have your Application(s) configured in Cloudflare. Refer to Cloudflare documentation if your tunnel isn't working.
* Assumes that you are using the settings in the `docker-compose.yaml` files for ports, names, etc.
* Assumes you have configured the upstream DNS servers in Pi-hole
* Assumes that you have [1Password CLI installed](https://1password.com/downloads/command-line/) on your host machine

### Networking

1. Make sure you are running your services in the same network as your `cloudfared` container is running in.
1. `cloudflared` will create a bridge network named `cloudflared`, so you might as well put your services in that network. The [Docker documentation on networking](https://docs.docker.com/config/containers/container-networking/) is very good. Check it out if this is new to you.
1. You specify the network in the `docker-compose.yaml` file, and you can check out the details in the [Networking in Compose](https://docs.docker.com/compose/networking/) documentation. If you use the code in this repo, you'll see the network is specified in the `docker-compose.yaml` file. The containers are named, so we can use their names instead of IP addresses.
1. You can see which networks are configured by running `docker network ls` and you should see `cloudflared` listed
1. You can get details about the network by running `docker network inspect cloudflared`
1. You can see details about the container network configuration by running `docker inspect <container_name>`

### Connectivity

1. Check connectivity from a client computer such as your laptop, and verify that you can connect to the Pi-hole web UI `http://<server-ip>:8080/admin/index.php`
1. To check connectivity from `cloudflared` to `pi-hole` container run `docker exec -it cloudflared sh -c 'wget http://<container-ip-of-pi-hole>'` (you'll want to clean up and delete the `index.html` file when you're done).
1. Check connectivity between containers: `docker exec cloudflared ping pihole -c2` or use the IP address `docker exec cloudflared ping 172.18.0.4` [See more excellent networking tips here](https://maximorlov.com/4-reasons-why-your-docker-containers-cant-talk-to-each-other/#:~:text=Do%20the%20containers%20share%20a,isolation%20features%20provided%20by%20Docker.)
1. Run commands from within the container. The `cloudflared` containers run `sh`, and the `nginx` and `pihole` containers use `bash`

### DNS

1. From the server host run `resolvectl status` to verify which DNS servers are configured. It should be the IP address of the `pi-hole` container in the `cloudflared` network, `127.0.0.1` or the IP address of the server.
1. The `docker-compose.yaml` files are configured to use the `pi-hole` for DNS. You can verify the configuration for the containers by running `docker inspect <container_name>`. Check the DNS settings match what you expect them to be.
1. From your client device, run `nslookup google.com <Pi-Hole Public IP Address>` and you should get a list of hosts

### Nginx Reverse Proxy

I had a devil of a time with getting this setup until I made the config file name `default.conf`. After that, the configuration worked. But if you have trouble you can try using these documents to help:

* [Managing Content and Configuration](https://docs.nginx.com/nginx/admin-guide/installing-nginx/installing-nginx-docker/#managing-content-and-configuration-files)
* [How to setup a Docker Nginx reverse proxy server example](https://www.theserverside.com/blog/Coffee-Talk-Java-News-Stories-and-Opinions/Docker-Nginx-reverse-proxy-setup-example)

### Rebuilding Containers

 `op run --env-file="../.env" -- docker-compose up --build --remove-orphans --force-recreate -d` from the server command line in the conatiner-directory will kill the container and force a rebuild.

## ToDo

* /etc/netplan/50-cloud-init.yaml file
* ~~Make use of [Docker Compose environment variables](https://docs.docker.com/compose/environment-variables/)~~ DONE
* ~~Make use of [Pi-hole environment variables](https://discourse.pi-hole.net/t/restoring-default-pi-hole-adlist-s/32323)~~
* ~~Configure [DoH with cloudflared](https://docs.pi-hole.net/guides/dns/cloudflared/)~~
* Add code scanning
* Deploy with Terraform
