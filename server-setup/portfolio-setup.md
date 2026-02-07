# Self-Hosted Nginx Portfolio w/ Secure Reverse Proxy

## 📖 Project Overview
This project hosts my personal professional portfolio using a containerized Nginx web server. The primary goal was to create a lightweight, self-hosted environment that strictly adheres to security best practices by isolating the application from the public internet.

Instead of exposing ports directly to the host's public interface, this deployment utilizes a **Reverse Proxy architecture** (Pangolin). The Nginx container is bound specifically to an internal Docker bridge network, ensuring that traffic can only reach the site through the designated proxy entry point.

## 🛠️ Tech Stack & Infrastructure
* **Containerization:** Docker (Nginx:Alpine image for minimal footprint)
* **Web Server:** Nginx
* **Reverse Proxy:** Pangolin (manages ingress and SSL termination)
* **OS:** Linux (Ubuntu Server)
* **Networking:** Custom internal bridge binding for traffic isolation

## 🔐 Security & Network Topology
One of the key challenges in self-hosting is "Port Leakage"—accidentally exposing administrative ports to the public WAN.

To solve this, I configured the container to bind **only** to the internal Docker bridge interface (`172.19.0.x`) rather than the public `0.0.0.0`.

* **Public Request:** `https://umusig.com` ➡️ Pangolin Proxy
* **Internal Routing:** Pangolin ➡️ `172.19.0.1:7373` (Docker Bridge)
* **Direct IP Access:** `http://[Server_Public_IP]:7373` ➡️ **Blocked/Refused**

## 🚀 Deployment

The service is deployed as a detached Docker container with a persistent volume map for live HTML updates.

```bash
docker run -d \
  --name portfolio \
  --restart unless-stopped \
  -p 172.19.0.1:7373:80 \
  -v /home/nathan/portfolio:/usr/share/nginx/html \
  nginx:alpine
