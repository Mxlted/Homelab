services:
  hawser:
    image: ghcr.io/finsys/hawser:latest
    container_name: hawser
    restart: unless-stopped
    ports:
      - "2376:2376"
    environment:
      TOKEN: TOKEN_HERE
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - hawser_stacks:/data/stacks
volumes:
  hawser_stacks:
