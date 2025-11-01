```yaml
services:
  watchtower:
    image: containrrr/watchtower
    container_name: watchtower
    environment:
      - TZ=America/New_York
      - WATCHTOWER_CLEANUP=true
      - WATCHTOWER_INCLUDE_STOPPED=true
      - WATCHTOWER_SCHEDULE=0 0 * * * * # runs every hour at minute 0
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    restart: unless-stopped
```
