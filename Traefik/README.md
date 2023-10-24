# Configure Traefik Proxy for Docker Containers

## Preparation for Docker Proxy Enviroment
1. Prepare Docker to use Corporate Proxy

```
> mkdir -p /etc/systemd/system/docker.service.d
> vi /etc/systemd/system/docker.service.d/http-proxy.conf
> cat /etc/systemd/system/docker.service.d/http-proxy.conf
[Service]
Environment="HTTP_PROXY=username:passw0rd@proxy.fqdn:8080"
Environment="HTTPS_PROXY=username:passw0rd@proxy.fqdn:8080"
Environment="NO_PROXY=*.socgen"

```

2. Create Docker Network for Traefik
```
> docker network create traefik
> docker network list
NETWORK ID     NAME      DRIVER    SCOPE
af15f82eecd1   traefik   bridge    local
```

## Docker-Compose Configuration - Traefik Proxy

| view      | url                                    |
|-----------|----------------------------------------|
| dashboard | https://hostname.fqdn:8080/dashboard/  |
| ping      | https://hostname.fqdn:8082/ping        |
| api       | https://hostname.fqdn:8080/api/http/routers <br> https://hostname.fqdn:8080/api/http/services <br> https://hostname.fqdn:8080/api/http/middlewares |

```
version: "3.3"

services:
  traefik:
    image: "traefik:latest"
    container_name: "traefik"
    command:
      #- "--log.level=DEBUG"
      - "--api.insecure=false"
      - "--api.dashboard=true"
      - "--ping=true"
      - "--ping.entryPoint=ping"
      - "--ping.manualrouting=true"
      - "--ping.terminatingStatusCode=204"
      - "--providers.docker=true"
      - "--providers.file.directory=/etc/traefik/dynamic_conf"
      - "--providers.docker.exposedbydefault=false"
      - "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"
      - "--entrypoints.traefik.address=:8080"
      - "--entrypoints.ping.address=:8082"
      - "--entrypoints.web.http.redirections.entrypoint.to=websecure"
      - "--entrypoints.web.http.redirections.entrypoint.scheme=https"
    networks:
      - traefik
    ports:
      - "443:443"
      - "80:80"
      - "8080:8080"
      - "8082:8082"
    restart: unless-stopped
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock:ro"
      - "/home/docker/traefik/dynamic_conf/dynamic_conf.yml:/etc/traefik/dynamic_conf/dynamic_conf.yml:ro"
      - "/home/docker/traefik/tls/cert.pem:/etc/ssl/certs/cert.pem:ro"
      - "/home/docker/traefik/tls/cert_priv.key:/etc/ssl/private/cert_priv.key:ro"
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.api.rule=Host(`hostname.fqdn `) && (PathPrefix(`/api`) || PathPrefix(`/dashboard`))"
      - "traefik.http.routers.api.service=api@internal"
      - "traefik.http.routers.api.entrypoints=traefik"
      - "traefik.http.routers.api.tls=true"
      - "traefik.http.routers.ping.rule=Host(`hostname.fqdn `) && PathPrefix(`/ping`)"
      - "traefik.http.routers.ping.service=ping@internal"
      - "traefik.http.routers.ping.entrypoints=ping"
      - "traefik.http.routers.ping.tls=true"

networks:
  traefik:
    external: true
```

## Traefik Dynamic Config
```
#/home/docker/traefik/dynamic_conf/dynamic_conf.yml
tls:
  certificates:
    - certFile: /etc/ssl/certs/cert.pem
      keyFile: /etc/ssl/private/cert_priv.key
```

## Docker-Compose Configuration - Web Application
```
version: "3.3"

services:
  whoami:
    image: "traefik/whoami"
    restart: unless-stopped
    deploy:
      replicas: 3
    networks:
      - traefik
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.whoami.rule=Host(`hostname.fqdn`)"
      - "traefik.http.routers.whoami.entrypoints=websecure"
      - "traefik.http.routers.whoami.tls=true"
 
networks:
  traefik:
    external: true
```


## Query Service Status

```
> curl  https://hostname.fqdn:8080/api/http/services/whoami-whoami@docker| jq ".serverStatus"
{
  http://172.19.0.3:80: "UP",
  http://172.19.0.4:80: "UP",
  http://172.19.0.5:80: "UP",
}
```
