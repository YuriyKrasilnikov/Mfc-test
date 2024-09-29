# traefik-swarm-example
```
version: '3.9'

networks:
  dockerproxy:
    internal: true
    attachable: false
    name: dockerproxy
  backend:
    internal: true
    attachable: true
    name: backend
  api:
    internal: false
    attachable: false
    name: api


services:
  traefik:
    depends_on:
      - dockerproxy
    image: "traefik:${TFKVER:-v2.11.8}"
    volumes:
      - /data/traefik/certs:/letsencrypt
    command:
      - --log.level=DEBUG
      - --accesslog=true

      - --providers.docker=true
      - --providers.docker.swarmmode=true
      - --providers.docker.network=backend
      - --providers.docker.endpoint=tcp://dockerproxy:2375
      - --providers.docker.exposedByDefault=false

      - --entrypoints.web.address=:80

      - --entrypoints.websecure.address=:443
      - --entrypoints.websecure.http.tls=true
      - --entrypoints.websecure.http.tls.certresolver=dev

      - --entrypoints.web.http.redirections.entrypoint.to=websecure
      - --entrypoints.web.http.redirections.entrypoint.scheme=https
      - --entrypoints.web.http.redirections.entrypoint.permanent=true

      - --certificatesresolvers.devdeeray.acme.tlschallenge=true
      - --certificatesresolvers.devdeeray.acme.storage=/letsencrypt/acme.json
    ports:
      - 80:80
      - 443:443
    deploy:
      placement:
        constraints:
          #- node.labels.type == app
          - node.role==balancer
      resources:
        # limits:
        #   cpus: "1.0"
        #   memory: "2G"
        reservations:
          cpus: "0.5"
        # memory: "1G"
    networks:
      - dockerproxy
      - backend
      - api
    logging:
      driver: json-file
      options:
        max-size: "15m"
        max-file: "5"

  dockerproxy:
    image: tecnativa/docker-socket-proxy:latest
    environment:
      # - LOG_LEVEL=info # debug,info,notice,warning,err,crit,alert,emerg
      ## Variables match the URL prefix (i.e. AUTH blocks access to /auth/* parts of the API, etc.).
      # 0 to revoke access.
      # 1 to grant access.
      ## Granted by Default
      - EVENTS=1
      - PING=1
      - VERSION=1
      ## Revoked by Default
      # Security critical
      - AUTH=0
      - SECRETS=0
      - POST=1 # Watchtower
      # Not always needed
      - BUILD=0
      - COMMIT=0
      - CONFIGS=0
      - CONTAINERS=1 # Traefik, portainer, etc.
      - DISTRIBUTION=0
      - EXEC=0
      - IMAGES=1 # Portainer
      - INFO=1 # Portainer
      - NETWORKS=1 # Portainer
      - NODES=0
      - PLUGINS=0
      - SERVICES=1 # Portainer
      - SESSION=0
      - SWARM=0
      - SYSTEM=0
      - TASKS=1 # Portainer
      - VOLUMES=1 # Portainer
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    deploy:
      placement:
        constraints:
          #- node.labels.type == app
          - node.role==balancer
      resources:
        # limits:
        #   cpus: "1.0"
        #   memory: "2G"
        reservations:
          cpus: "1"
        # memory: "1G"
    networks:
      - dockerproxy
    logging:
      driver: json-file
      options:
        max-size: "15m"
        max-file: "5"

  whoami:
    image: traefik/whoami
    deploy:
      labels:
        - traefik.enable=true
        - traefik.http.services.whoami.loadbalancer.server.port=80
        - traefik.http.routers.whoami.rule=Host(whoami.local)
        - traefik.http.routers.whoami.entrypoints=web
      replicas: 2
    networks:
      - backend
```

For test `curl -H "Host: whoami.deeray.local" <YOUR_IP>`
