# Mfc-test
```
version: '3.7'

services:
  traefik:
    image: traefik:v2.3
    command:
      - --api.insecure=true
      - --providers.docker=true
      - --providers.docker.endpoint=tcp://dockerproxy:2375
      - --providers.docker.swarmMode=true
      - --providers.docker.exposedByDefault=false
      - --entrypoints.web.address=:80
    ports:
      - 80:80
      - 8080:8080
    deploy:
      placement:
        constraints:
          - node.role==manager

  dockerproxy:
    image: tecnativa/docker-socket-proxy
    environment:
      CONTAINERS: 1
      NETWORKS: 1
      SERVICES: 1
      SWARM: 1
      TASKS: 1
      VOLUMES: 1
      SECRETS: 1
      CONFIGS: 1
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    deploy:
      placement:
        constraints:
          - node.role==manager

  whoami:
    image: containous/whoami
    labels:
      - "traefik.http.routers.whoami.rule=PathPrefix(`/`)"
      - "traefik.http.routers.whoami.entrypoints=web"
    deploy:
      replicas: 2
```
