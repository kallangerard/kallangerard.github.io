---
title: 'How to Use Traefik 2 With Home Assistant'
date: 2020-10-13T13:10:41+08:00
draft: true
---

# How to use Traefik v2 with Home Assistant and Docker

When I started using Traefik v2, I struggled to find a useful guide to use it with Home Assistant. Here's my attempt to fill that gap.

<!--more-->

To attempt this you'll want a working understanding of how reverse proxies are used, as well as experience with Docker. You'll also require a domain name.

The main benefits of Traefik for me have been:

- **Service Discovery:** Traefik can use metadata from your Docker services to discover those services and dynamically configure itself.
- **SSL Termination and Automatic certificate generation:** Using Let's Encrypt you can get self-renewing certificates out of the box, including for use inside your own network.
- **Middleware:** Hand off authentication to another service, set headers, redirect HTTP to HTTPS etc.

```yaml
# docker-compose.yaml
---
version: "2.3"

services:

  traefik:
      image: traefik:v2.2.1
      container_name: traefik
      ports:
        - 80:80
        - 443:443
      networks:
        - default
        - backend
      environment:
        - CLOUDFLARE_DNS_API_TOKEN=${CLOUDFLARE_DNS_API_TOKEN}
      volumes:
        - /var/run/docker.sock:/var/run/docker.sock:ro
        - traefik:/letsencrypt
      command:
	      - --entrypoints.web.address=:80
        - --entrypoints.websecure.address=:443
        - --providers.docker
        - --providers.docker.exposedbydefault=false
        - --providers.docker.endpoint=unix://var/run/docker.sock
        # network name is <projectname>_backend
        - --providers.docker.network=docker_backend
        # certificatesresolver.<any_name>.acme
        - --certificatesresolvers.cloudflare.acme.dnschallenge.provider=cloudflare
        - --certificatesresolvers.cloudflare.acme.storage=/letsencrypt/acme.json
        - --certificatesresolvers.cloudflare.acme.dnschallenge.resolvers=1.1.1.1:53,8.8.8.8:53
      labels:
        - "traefik.enable=true"
        # global redirect to https
        - "traefik.http.routers.redirs.entrypoints=web"
        - "traefik.http.routers.redirs.rule=hostregexp(`{host:.+}`)"
        # HTTP>HTTPS REDIRECT
        - "traefik.http.routers.redirs.middlewares=redirect-to-https"
        - "traefik.http.middlewares.redirect-to-https.redirectscheme.scheme=https"
      restart: always

  homeassistant:
    image: homeassistant/home-assistant:0.113.2
    container_name: homeassistant
    networks:
      - backend
    environment:
      - PGID=${DOCKER_PGID}
      - PUID=${DOCKER_PUID}
      - TZ=${TIME_ZONE}
    volumes:
      - homeassistant:/config
      - /etc/localtime:/etc/localtime:ro
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.homeassistant.entrypoints=websecure"
      - "traefik.http.routers.homeassistant.rule=Host(`homeassistant.${DOMAIN_NAME}`)"
      - "traefik.http.routers.homeassistant.tls=true"
      - "traefik.http.routers.homeassistant.tls.certresolver=cloudflare"
      - "traefik.http.services.homeassistant.loadbalancer.server.port=8123"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8123"]
      interval: 2m
      timeout: 10s
      retries: 3
      start_period: 40s
    restart: always

networks:
  default:
  backend:

volumes:
  traefik:
  homeassistant:
```

_Here's a complete working example for those of you who are almost there but just want a quick reference guide. Please note I'm using Home Assistant inside a docker network and not on host_mode. For device discovery in this mode I recommend using a second remote home assistant instance which I'll cover in another article._

Let's start from the top. I'm using Docker Compose version 2. If you're not using Docker Compose I highly recommend it over `docker run` commands.

```yaml
traefik:
    # Make sure to use a specific image version.
    # Your Application Proxy is not something you want pulling latest.
		...
		# You will want at least two networks.
		# The default network services run by default, and where HTTP/HTTPS exposed.
		# And one or more backend networks for your services.
    networks:
      - default
      - backend
```

```yaml
volumes:
  - /var/run/docker.sock:/var/run/docker.sock:ro
  - traefik:/letsencrypt
```

Here we mount the docker socket to Traefik with read only access. This allows Traefik to read the metadata of your containers for service discovery. We also use a volume for traefik to store certificates.

```yaml
command:
  - --entrypoints.web.address=:80
  - --entrypoints.websecure.address=:443
  - --providers.docker
  - --providers.docker.exposedbydefault=false
  - --providers.docker.endpoint=unix://var/run/docker.sock
  # network name is <projectname>_backend
  - --providers.docker.network=docker_backend
  - --certificatesresolvers.cloudflare.acme.dnschallenge.provider=cloudflare
  - --certificatesresolvers.cloudflare.acme.storage=/letsencrypt/acme.json
  - --certificatesresolvers.cloudflare.acme.dnschallenge.resolvers=1.1.1.1:53,8.8.8.8:53
```

Here we're using a custom command to configure traefik. I prefer this over using configuration files as it keeps it all within one place. Docker Compose will combine this entire list into a single line, but it's a lot more readable this way.

There's a few ways to configure traefik, whichever method you prefer.

**Pluralisation and capitalisation are different for each configuration mode.**

```yaml
# docker-compose.yaml
command:
  - --entrypoints.web.address=:80
  - --entrypoints.websecure.address=:443

# traefik.yaml
entryPoints:
  web:
    address: ":80"
  websecure:
    address: ":443"

# traefik.toml
[entryPoints.web]
address = ":80"
[entryPoints.websecure]
address = ":443"
```

These are all equivalent.

**Entrypoints** define the outside edge of your traefik service. For standard HTTP you'll need port 80 and for HTTPS you'll need 443.

```yaml
- --entrypoints.web.address=:80
- --entrypoints.websecure.address=:443
```

**Providers** define where traefik will look for services.

```yaml
- --providers.docker
- --providers.docker.exposedbydefault=false
- --providers.docker.endpoint=unix://var/run/docker.sock
- --providers.docker.network=docker_backend
```

We need one flag to declare docker being used as a provider, another to define the endpoint, as well as `exposedbydefault=false` so that we have to explicitly tell traefik to enable services on each container. We also set the network to match what is used by your service containers, to avoid having to set it for every individual container.

The name of the network is the **absolute** network name, which by default is the name of the folder your docker-compose.yaml file lives and the name of the network. To see your network names run `docker network ls`. In my case it's `docker_backend`.

**Certificate Resolvers** are used for the automatic generation of your certificates for HTTPS. Check traefik's documentation for your exact configuration requirements. [https://doc.traefik.io/traefik/https/acme/](https://doc.traefik.io/traefik/https/acme/)

```yaml
# Since there is no inline assignment, Docker Compose will pull this variable
# from the environment where you run docker-compose up
environment:
  - CLOUDFLARE_DNS_API_TOKEN
command:
	# ...
	- --certificatesresolvers.cloudflare.acme.dnschallenge.provider=cloudflare
	- --certificatesresolvers.cloudflare.acme.storage=/letsencrypt/acme.json
	- --certificatesresolvers.cloudflare.acme.dnschallenge.resolvers=1.1.1.1:53,8.8.8.8:53
```

I'm using a dns challenge process to allow Let's Encrypt to prove I control my domain.

When a service is defined with a host name, Traefik will kick off the certificate generation process.

Traefik will call Let’s Encrypt and receive a token, and then create a TXT record derived from that token and your account key with your dns provider at \_acme-challenge.<YOUR_DOMAIN>. Traefik will then check the DNS record for your domain until the token resolves. Afterwards Let’s Encrypt will validate the DNS on it's end and hand over a certificate to Traefik.

I'm manually specifying the DNS servers to check the TXT record with `certificatesresolvers.cloudflare.acme.dnschallenge.resolvers` because my internal DNS would interfer with the check.

Traefik will then take that certificate and store it permanently in the location defined in `certificatesresolvers.cloudflare.acme.storage`.

The main advantage here is, unlike a HTTP challenge, your Traefik instance does not need to be reachable from the internet at all. As long as it can make an outgoing connection to your DNS provider and Let's Encrypt you're good to keep your services behind a VPN and firewall at all times.

Finally we're defining the resolvers TODO: Describe resolvers

## Redirecting HTTP to HTTPS (optional)

```yaml
labels:
  - 'traefik.enable=true'
  # global redirect to https
  - 'traefik.http.routers.redirs.entrypoints=web'
  - 'traefik.http.routers.redirs.rule=hostregexp(`{host:.+}`)'
  # HTTP>HTTPS REDIRECT
  - 'traefik.http.routers.redirs.middlewares=redirect-to-https'
  - 'traefik.http.middlewares.redirect-to-https.redirectscheme.scheme=https'
```

Here's a label definition under Traefik to create some rules. Note this doesn't actually enable communication to the Traefik container itself, we're just using it to define some global rules as oppose to using an external file.

Traefik is comprised of entrypoints, routers, middleware and services. We create a http router rule that listens on the web entrypoint (port 80), with a regex express that applies to _all_ host names.

Backticks ``` are required to define a host rule, not single quotes

The router is called 'redirs'.

Then we create a redirection to forward to a middleware called 'redirect-to-https'. Which will redirect the incoming request to https.

## Defining Home Assistant Rules

```yaml
  homeassistant:
    ...
    networks:
      - backend
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.homeassistant.entrypoints=websecure"
      - "traefik.http.routers.homeassistant.rule=Host(`homeassistant.${DOMAIN_NAME}`)"
      - "traefik.http.routers.homeassistant.tls=true"
      - "traefik.http.routers.homeassistant.tls.certresolver=cloudflare"
      - "traefik.http.services.homeassistant.loadbalancer.server.port=8123"
      - "traefik.http.services.homeassistant.loadbalancer.sticky=true"
      - "traefik.http.services.homeassistant.loadbalancer.sticky.cookie.name=homeassistant"
```

You'll need to set up your Home Assistant definition in Docker Compose to your particular circumstances, which is outside the scope of this article. But to enable Traefik we need to provide at least the following rules.

- The container needs to use the backend network

```yaml
networks:
  - backend
```

- We tell Traefik to use this container

```yaml
- 'traefik.enable=true'
```

- Allow HTTP access to Home Assistant on the websecure endpoint (port 443), and define a rule to match. In this case I'm using a subdomain of homeassistant with a domain name I own.

```yaml
- 'traefik.http.routers.homeassistant.entrypoints=websecure'
- 'traefik.http.routers.homeassistant.rule=Host(`homeassistant.${DOMAIN_NAME}`)'
```

- Request a certificate for HTTPS from Let's Encrypt

```yaml
- 'traefik.http.routers.homeassistant.tls=true'
- 'traefik.http.routers.homeassistant.tls.certresolver=cloudflare'
```

- If they're authored properly, Traefik will be able to get this information from the docker container itself without you having to define it manually. But unfortunately that's not the case here, so we have to define the port inside the container for Traefik to forward to.

```yaml
- 'traefik.http.services.homeassistant.loadbalancer.server.port=8123'
```

Note that every service in Traefik has a load balancer, even if there is only one server.

```yaml
networks:
  default:
  backend:
```

Finally we just need to define the networks used by Traefik. As noted earlier, when defining the network name in traefik configuration you must use the absolute network name, which by default is `<docker-compose-project-name>_<network_name>`.
