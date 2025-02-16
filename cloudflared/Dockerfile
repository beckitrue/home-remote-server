# syntax=docker/dockerfile:1 

# Dockerfile to build a cloudflared proxy-dns tunnel image using 
# alpine mininimal as the base image
# Copy the cloudflared binary to the image
# chmod  +x cloudflared
# add a non-root user to run the cloudflared binary
# and run the cloudflared binary with the proxy-dns arguments
# other Cloudflare endpoints can be used as the upstream DNS server
# by setting the TUNNEL_DNS_UPSTREAM environment variable
# https://developers.cloudflare.com/1.1.1.1/infrastructure/network-operators/#available-endpoints

# Use the official Alpine image for a minimal base
FROM alpine:3.21.3

# Download and install Cloudflared
RUN wget https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64 -O /usr/bin/cloudflared
RUN chmod +x /usr/bin/cloudflared

# Create a non-root user to run Cloudflared
RUN adduser -D -g '' -H cloudflared
RUN chown cloudflared:cloudflared /usr/bin/cloudflared

USER cloudflared

# Set the DNS port and upstream for proxy-dns
ENV TUNNEL_DNS_PORT="5053"
ENV TUNNEL_DNS_UPSTREAM="https://security.cloudflare-dns.com/dns-query"
ENV TUNNEL_DNS_ADDRESS="0.0.0.0"

# Run the proxy-dns subcommand with options by default
CMD /usr/bin/cloudflared proxy-dns --port $TUNNEL_DNS_PORT --upstream $TUNNEL_DNS_UPSTREAM --address $TUNNEL_DNS_ADDRESS

