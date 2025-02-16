# syntax=docker/dockerfile:1 

# Dockerfile to build a cloudflared image using alpine mininimal image
# and copy the cloudflared binary to the image
# chmod  +x cloudflared
# add a non-root user to run the cloudflared binary
# and run the cloudflared binary with the proxy-dns argument

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

# Run Cloudflared with the proxy-dns command by default
CMD /usr/bin/cloudflared proxy-dns --port $TUNNEL_DNS_PORT --upstream $TUNNEL_DNS_UPSTREAM

