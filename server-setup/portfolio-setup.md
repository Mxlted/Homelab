# Self-Hosted Nginx Portfolio w/ Secure Reverse Proxy

## Project Overview

This project hosts my personal professional portfolio using a lightweight, containerized **Nginx** web server. The primary objective was to build a minimal self-hosted deployment that follows strong security best practices by preventing accidental public exposure of internal services.

Instead of binding the web server directly to the host’s public interface, this setup uses a **Reverse Proxy architecture** (Pangolin) to manage all external access, SSL termination, and controlled ingress routing.

---

## Tech Stack & Infrastructure

- **Containerization:** Docker (`nginx:alpine` for minimal footprint)
- **Web Server:** Nginx
- **Reverse Proxy:** Pangolin (handles ingress + HTTPS termination)
- **OS:** Ubuntu Server (Linux)
- **Networking:** Internal Docker bridge binding for isolation

---

## Security & Network Topology

One of the most common risks in self-hosting is **port leakage**, where application ports unintentionally become accessible from the public WAN.

To eliminate this risk, the portfolio container is bound **only** to an internal Docker bridge interface (`172.19.0.x`) instead of `0.0.0.0`. This ensures that the site can only be reached through the reverse proxy.

### Request Flow

- **Public Request:** `https://umusig.com` ➝ Pangolin Proxy  
- **Internal Routing:** Pangolin ➝ `172.19.0.1:7373`  
- **Direct Access Attempt:** `http://[Public_IP]:7373` ➝ **Blocked/Refused**

This design guarantees the container is not directly exposed to the internet.

---

## Domain Canonicalization (www Redirect)

To ensure consistent access and prevent DNS mismatch issues between the apex domain and subdomain, I configured Nginx to enforce a canonical host:

- Requests to `umusig.com` automatically redirect to `www.umusig.com`

This avoids split-site behavior and ensures all traffic resolves correctly through the proxy.

### Nginx Configuration (`nginx.conf`)

```nginx
server {
  listen 80;
  server_name umusig.com;
  return 301 http://www.umusig.com$request_uri;
}

server {
  listen 80;
  server_name www.umusig.com;
  root /usr/share/nginx/html;
  index index.html;
}
```

---

## Deployment

The portfolio runs as a detached Docker container with:

- A static HTML volume mount
- A custom Nginx configuration override
- Read-only filesystem enforcement for additional security

```bash
docker run -d \
  --name portfolio \
  --restart unless-stopped \
  -p 172.19.0.1:7373:80 \
  -v /home/nathan/portfolio:/usr/share/nginx/html:ro \
  -v /home/nathan/portfolio/nginx.conf:/etc/nginx/conf.d/default.conf:ro \
  nginx:alpine
```

### Key Security Benefits

- Container traffic is restricted to an internal bridge interface
- No direct WAN access to the exposed port
- Reverse proxy is the only ingress point
- Static content mounted as **read-only**
- Canonical domain redirect prevents hostname inconsistencies

---

## Summary

This deployment demonstrates a secure and production-style approach to self-hosting a static portfolio:

- Minimal Docker footprint  
- Strict network isolation  
- Reverse proxy-controlled ingress  
- Hardened container mounts  
- Clean domain routing via Nginx redirects  
