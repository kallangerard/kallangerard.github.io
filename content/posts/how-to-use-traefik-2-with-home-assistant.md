---
title: 'How to Use Traefik v2 With Home Assistant'
date: 2020-10-13T13:10:41+08:00
draft: false
---

When I started using Traefik v2, I struggled to find a useful guide to use it with Home Assistant. Here's my attempt to fill that gap.

<!--more-->

To attempt this you'll want a working understanding of how reverse proxies are used, as well as experience with Docker. You'll also require a domain name.

The main benefits of Traefik for me have been:

- **Service Discovery:** Traefik can use metadata from your Docker services to discover those services and dynamically configure itself.
- **SSL Termination and Automatic certificate generation:** Using Let's Encrypt you can get self-renewing certificates out of the box, including for use inside your own network.
- **Middleware:** Hand off authentication to another service, set headers, redirect HTTP to HTTPS etc.

_Here's a complete working example for those of you who are almost there but just want a quick reference guide. Please note I'm using Home Assistant inside a docker network and not on host_mode. For device discovery in this mode I recommend using a second remote home assistant instance which I'll cover in another article._

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

## Defining Traefik in Docker Compose

Let's start from the top.

You will need at least two networks. Here I'm using the default, and another for the backend services.

_Make sure to use a specific image version. Your Application Proxy is not something you want pulling latest_

```yaml
traefik:
    image: traefik:v2.2.1
		...
    networks:
      - default
      - backend
```

Traefik will require read only access to your docker socket. This allows Traefik to read the metadata of your containers for service discovery. We also use a volume for traefik to store certificates.

```yaml
volumes:
  - /var/run/docker.sock:/var/run/docker.sock:ro
  - traefik:/letsencrypt
```

I use a custom docker run command to configure Traefik. I prefer this over using configuration files as it keeps it all within one place. In addition, when you make a change Docker Compose will restart Traefik for you.

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

There's a few ways to configure Traefik, you can use whichever method you prefer, they are all equivalent.

_Pluralisation and capitalisation are different for each configuration mode._

```yaml
# docker-compose.yaml
command:
  - --entrypoints.web.address=:80
  - --entrypoints.websecure.address=:443
```

```yaml
# traefik.yaml
entryPoints:
  web:
    address: ':80'
  websecure:
    address: ':443'
```

```toml
# traefik.toml
[entryPoints.web]
address = ":80"
[entryPoints.websecure]
address = ":443"
```

## Entrypoints

Entrypoints define the outside edge of your Traefik service. For standard HTTP you'll need port 80 and for HTTPS you'll need 443.

```yaml
- --entrypoints.web.address=:80
- --entrypoints.websecure.address=:443
```

## Providers

Providers define where Traefik will look for services.

We need one flag to declare docker being used as a provider.

```yaml
- --providers.docker
```

To make sure Traefik doesn't try to expose every Docker service by default, we declare

```yaml
- --providers.docker.exposedbydefault=false
```

We also set the network to match what is used by your service containers, to avoid having to set it for every individual container.

The name of the network is the **absolute** network name, which by default is the name of the folder where your docker-compose.yaml file lives and the name of the network. To confirm your network names run `docker network ls`. In my case it's `docker_backend`.

```yaml
- --providers.docker.network=docker_backend
```

## Certificate Resolvers

Certificate Resolvers are used for the automatic generation of your certificates for HTTPS. Check traefik's documentation for your exact configuration requirements. [https://doc.traefik.io/traefik/https/acme/](https://doc.traefik.io/traefik/https/acme/)

_Since there is no inline assignment, Docker Compose will pull this variable
from the environment when you run `docker-compose up`_

```yaml

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

I'm manually specifying the DNS servers to check the TXT record with `certificatesresolvers.cloudflare.acme.dnschallenge.resolvers` because my internal DNS would interfer with this check.

Traefik will then take that certificate and store it permanently in the location defined in `certificatesresolvers.cloudflare.acme.storage`.

The main advantage here is, unlike a HTTP challenge, your Traefik instance does not need to be reachable from the internet at all. As long as it can make an outgoing connection to your DNS provider and Let's Encrypt you're good to keep your services behind a VPN and firewall at all times.

## Redirecting HTTP to HTTPS (optional)

Traefik is comprised of entrypoints, routers, middleware and services.
We create a http router rule, arbitrarily named `redirs`, to listen on the web entrypoint (port 80), with a regex expression that applies to _all_ host names. So effectively every HTTP client request that hits Traefik on port 80 will be handled by this router.

_Backticks ` are required to define a host rule, not single quotes '_

```yaml
labels:
  - 'traefik.enable=true'
  # global redirect to https
  - 'traefik.http.routers.redirs.entrypoints=web'
  - 'traefik.http.routers.redirs.rule=hostregexp(`{host:.+}`)'
```

Then we define a piece of middleware, arbitrarily named `redirect-to-https`, to redirect incoming request to https.

```yaml
# HTTP>HTTPS REDIRECT
- 'traefik.http.routers.redirs.middlewares=redirect-to-https'
- 'traefik.http.middlewares.redirect-to-https.redirectscheme.scheme=https'
```

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

The container needs to use the backend network

```yaml
networks:
  - backend
```

We tell Traefik to use this container

```yaml
- 'traefik.enable=true'
```

Allow HTTP access to Home Assistant on the websecure endpoint (port 443), and define a rule to match. In this case I'm using a subdomain of homeassistant with a domain name I own.

```yaml
- 'traefik.http.routers.homeassistant.entrypoints=websecure'
- 'traefik.http.routers.homeassistant.rule=Host(`homeassistant.${DOMAIN_NAME}`)'
```

Request a certificate for HTTPS from Let's Encrypt, using the certsresolver `cloudflare` we defined earlier.

```yaml
- 'traefik.http.routers.homeassistant.tls=true'
- 'traefik.http.routers.homeassistant.tls.certresolver=cloudflare'
```

If a container is authored properly, Traefik will be able to get this information from the docker container itself without you having to define it manually. But unfortunately that's not the case here, so we have to define the port inside the container for Traefik to forward to.

_Every service in Traefik has a load balancer, even if there is only one upstream service_

```yaml
- 'traefik.http.services.homeassistant.loadbalancer.server.port=8123'
```

Finally we just need to define the networks used by Traefik. As noted earlier, when defining the network name in traefik labels or configuration, you must use the absolute network name, which by default is `<docker_compose_project_name>_<network_name>`.

```yaml
networks:
  default:
  backend:
```

## DNS

Now the final catch is making sure you and your clients can reach your Traefik instance. Depending on your network configuration you'll have to do one of the following:

### Internal Access Only

You will need to use an internally managed DNS. I use my router to create a dnsmasq entry for my domain name, with the IP address of the server hosting Traefik. Be sure to include your subdomains or use a subdomain wildcard.

### External Web Access

You'll need to forward port 80 and 443 to your Traefik server. And you'll probably need to enable a internal NAT reflection so your internal clients will be redirected appropriately.

You will also need to make sure your dns record with your DNS provider resolves to your WAN ip address.

_These are both highly dependent on your network configuration. I wouldn't attempt these until you have a reasonable understanding of what you're doing_

## Bringing it up

Now we're finally ready to bring up your services. `docker-compose up -d` should bring up your Traefik and Home Assistant services up, and Traefik will read it's own and Home Assistant's labels. And now you can reach your Home Assistant instance with `https://homeassistant.domain.tld`.

## Problems

There's a lot of moving parts here, so don't stress if you have problems getting it to work at first. If you're not comfortable at debugging Docker services you might need to take a step back and bring up your skills there. Also make sure to read the documentation for Docker Compose, Docker and Traefik.

If you have any questions just leave a comment and I'm happy to help.
