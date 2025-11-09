```yaml
services:
  qbittorrent:
    hostname: qbittorrent.internal
    container_name: qbittorrent
    image: ghcr.io/hotio/qbittorrent
    restart: unless-stopped
    ports:
      - 8080:8080
    environment:
      - PUID=0
      - PGID=0
      - UMASK=002
      - TZ=America/New_York
      - WEBUI_PORTS=8080/tcp,8080/udp
      - VPN_ENABLED=true
      - VPN_CONF=wg0backup
      - VPN_PROVIDER=generic
      - VPN_LAN_NETWORK=192.168.1.0/24
      - VPN_LAN_LEAK_ENABLED=false
      - VPN_EXPOSE_PORTS_ON_LAN=
      - VPN_AUTO_PORT_FORWARD=true
      - VPN_AUTO_PORT_FORWARD_TO_PORTS=5175
      - VPN_FIREWALL_TYPE=auto
      - VPN_HEALTHCHECK_ENABLED=false
      - VPN_NAMESERVERS=wg
      - PRIVOXY_ENABLED=false
    cap_add:
      - NET_ADMIN
    sysctls:
      - net.ipv4.conf.all.src_valid_mark=1
      - net.ipv6.conf.all.disable_ipv6=1
    volumes:
      - /mnt/nas/configs/qbittorrent:/config
      - /mnt/nas/configs/qbittorrent/cache:/app/qBittorrent/cache
      - /mnt/nas/configs/qbittorrent/data:/app/qBittorrent/data
      - /mnt/nas/media:/media
  prowlarr:
    image: lscr.io/linuxserver/prowlarr:latest
    container_name: prowlarr
    environment:
      - TZ=America/New_York
      - PUID=0
      - PGID=0
    volumes:
      - /mnt/nas/configs/prowlarr:/config
      - /mnt/nas/media/:/media/
    ports:
      - 9696:9696
    restart: unless-stopped
  radarr:
    image: lscr.io/linuxserver/radarr:latest
    container_name: radarr
    environment:
      - TZ=America/New_York
      - PUID=0
      - PGID=0
    volumes:
      - /mnt/nas/configs/radarr:/config
      - /mnt/nas/media:/media
    ports:
      - 7878:7878
    restart: unless-stopped
  sonarr:
    image: lscr.io/linuxserver/sonarr:latest
    container_name: sonarr
    environment:
      - TZ=America/New_York
      - PUID=0
      - PGID=0
    volumes:
      - /mnt/nas/configs/sonarr:/config
      - /mnt/nas/media:/media
    ports:
      - 8989:8989
    restart: unless-stopped
  flaresolverr:
    image: ghcr.io/flaresolverr/flaresolverr:latest
    container_name: flaresolverr
    environment:
      - LOG_LEVEL=info
      - LOG_HTML=false
      - CAPTCHA_SOLVER=none
      - TZ=America/New_York
    ports:
      - 8191:8191
    restart: unless-stopped
  cross-seed:
    image: ghcr.io/cross-seed/cross-seed:6
    container_name: cross-seed
    hostname: cross-seed.internal
    user: 0:0
    ports:
      - 2468:2468
    volumes:
      - /mnt/nas/configs/crossseed:/config
      - /mnt/nas/media:/media
    command: daemon
    restart: unless-stopped
networks: {}
```

