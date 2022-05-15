---
title: Docker
sidebar_label: Docker setup
sidebar_position: 2
---

This guide expects some previous docker knowledge and a working environment.

## docker-compose

`docker-compose` for autobrr. Modify if running with unRAID or setting up with Portainer.

* Logging is optional
* Host port mapping might need to be changed to not collide with other apps
* Change `BASE_DOCKER_DATA_PATH` to match your setup. Can be simply `./data`
* Set custom network if needed

```yaml title="docker-compose.yml"
version: "3.7"

services:
  autobrr:
    container_name: autobrr
    image: ghcr.io/autobrr/autobrr:latest
    restart: unless-stopped
    #logging:
    #  driver: json-file
    #  options:
    #    max-file: ${DOCKERLOGGING_MAXFILE}
    #    max-size: ${DOCKERLOGGING_MAXSIZE}
    environment:
      - PUID=${PUID}
      - PGID=${GUID}
      - TZ=${TZ}        
    volumes:
      - ${BASE_DOCKER_DATA_PATH}/autobrr/config:/config
    ports:
      - 7474:7474
```

### Config

You can either let autobrr create the config itself at startup, or create one manually. Note! Default port was changed from 8989 to 7474.

If done manually, here's a good start. Put this in the directory in the volume map, like `./data`.

Create data dir and config `mkdir data && touch data/config.toml`.

```toml title="config.toml"
# config.toml

# Hostname / IP
#
# Default: "localhost"
#
host = "0.0.0.0"

# Port
#
# Default: 7474
#
port = 7474

# Base url
# Set custom baseUrl eg /autobrr/ to serve in subdirectory.
# Not needed for subdomain, or by accessing with the :port directly.
#
# Optional
#
#baseUrl = "/autobrr/"

# autobrr logs file
# If not defined, logs to stdout
#
# Optional
#
#logPath = "log/autobrr.log"

# Log level
#
# Default: "TRACE"
#
# Options: "ERROR", "DEBUG", "INFO", "WARN", "TRACE"
#
logLevel = "TRACE"

# Session secret
# Can be generated by running: head /dev/urandom | tr -dc A-Za-z0-9 | head -c16
sessionSecret = "secret-session-key"
```

### Config options

* `host`: Default: `0.0.0.0`. If running a reverse-proxy on the host, you could use `127.0.0.1` and have the portmap be `127.0.0.1:7474:7474` and let the RP handle the rest.
* `port`: If port already in use then change to a free one.
* `baseUrl`: **`OPTIONAL`** It supports running on both the root url and in a subpath, as well as subdomain. Uncomment if needed.
* `logPath`: **`OPTIONAL`** It can be useful to log to file, but probably better to let docker handle this.
* `logLevel`: Choose how much log output you want to see. Needs a restart to take effect.
* `sessionSecret`: Used for session cookies. Change to something more random like a `UUID`.

## Initial start

1. Start the container `docker-compose up -d`
2. Exec into the container with `docker exec -it autobrr sh`
3. Run the create-user command: `autobrrctl --config /config create-user <USERNAME>`. Then exit out with `exit`
4. Now you should be able to login with the newly created user.

## Reverse proxies

### Traefik

Traefik setup to run on subdomain.

* Needs an `.env` file with `DOMAIN` set, like `DOMAIN=something.local`
* Expects an externally created network called `proxy`
* Expects two `entryPoints`: `http` going to `:80` and `https` going to `:443`
* Expects a `certificateResolver` called `letsencrypt`

Your config may be different so change accordingly.

```yaml title="docker-compose.yml"
version: "3.7"

networks:
    proxy:
    external: true

services:
  autobrr:
    image: ghcr.io/autobrr/autobrr:latest
    container_name: autobrr
    restart: unless-stopped
    networks:
      - proxy
    volumes:
      - ./data:/config
    labels:
      - "traefik.enable=true"
      - "traefik.docker.network=proxy"

      - "traefik.http.middlewares.redirect-https.redirectScheme.scheme=https"
      - "traefik.http.middlewares.redirect-https.redirectScheme.permanent=true"

      - "traefik.http.routers.autobrr-https.rule=Host(`autobrr.$DOMAIN`)"
      - "traefik.http.routers.autobrr-https.entrypoints=https"
      - "traefik.http.routers.autobrr-https.tls=true"
      - "traefik.http.routers.autobrr-https.tls.certresolver=letsencrypt"
      - "traefik.http.routers.autobrr-https.service=autobrr"
      - "traefik.http.routers.autobrr-http.rule=Host(`autobrr.$DOMAIN`)"
      - "traefik.http.routers.autobrr-http.entrypoints=http"
      - "traefik.http.routers.autobrr-http.middlewares=redirect-https"
      - "traefik.http.routers.autobrr-http.service=autobrr"
      - "traefik.http.services.autobrr.loadbalancer.server.port=7474"
```

### Swag

A basic `swag` config for running on subdomain.

```conf
server {
    listen 443 ssl;
    listen [::]:443 ssl;

    server_name autobrr.*;

    include /config/nginx/ssl.conf;

    client_max_body_size 0;

    # enable for ldap auth, fill in ldap details in ldap.conf
    #include /config/nginx/ldap.conf;

    # enable for Authelia
    #include /config/nginx/authelia-server.conf;

    location / {
        # enable the next two lines for http auth
        #auth_basic "Restricted";
        #auth_basic_user_file /config/nginx/.htpasswd;

        # enable the next two lines for ldap auth
        #auth_request /auth;
        #error_page 401 =200 /ldaplogin;

        # enable for Authelia
        #include /config/nginx/authelia-location.conf;

        include /config/nginx/proxy.conf;
        include /config/nginx/resolver.conf;
        include /config/nginx/allowdeny.conf;
        set $upstream_app autobrr;
        set $upstream_port 7474;
        set $upstream_proto http;
        proxy_pass $upstream_proto://$upstream_app:$upstream_port;

    }
}
```