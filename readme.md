# Traefik 2 config


This is a resuseable traefik config for usage on a vServer using docker-compose.
It uses:
 - Traefik 2.2
 - docker-compose
 - Let's encrypt

## Table of content

* [Production setup](#production-setup)
  + [Setting up traefik](#setting-up-traefik)
  + [Traefik dashboard](#traefik-dashboard)
  + [Connect docker-compose service to reverse-proxy](#connect-docker-compose-service-to-reverse-proxy)
* [Setup for local development](#setup-for-local-development)
  + [Setting up traefik](#setting-up-traefik-1)
  + [Traefik dashboard](#traefik-dashboard-1)
  + [Connect docker-compose service to reverse-proxy](#connect-docker-compose-service-to-reverse-proxy-1)
* [Credits](#credits)
* [License](#license)

## Production setup

### Setting up traefik

1. Clone repository
   ```bash
   git clone https://github.com/korridor/reverse-proxy-docker-traefik.git
   cd reverse-proxy-docker-traefik
   ```
2. Copy default config  
   ```bash
   cp docker-compose.prod.yml docker-compose.yml
   cp -r configs-prod configs
   echo "{}" > certificates/acme.json
   chmod 600 certificates/acme.json
   ```
3. Replace domain for dashboard (`reverse-proxy.somedomain.com` in `configs/dynamic/dashboard.yml`)
   ```yaml
   http:
     routers:
       traefik:
         rule: Host(`reverse-proxy.somedomain.com`)
         # ...
       traefik-http-redirect:
         rule: Host(`reverse-proxy.somedomain.com`)
         # ...
   ```
4. Replace password for admin account (in `configs/dynamic/dashboard.yml`) 
    ```yaml
   http:
     # ...
     middlewares:
       dashboardauth:
         basicAuth:
           users:
             - "user1:$2y$05$/x10KYbrHtswyR8POT.ny.H4fFd1n.0.IEiYiestWzE1QFkYIEI3m"
    ```  
     - You can use a website like [this](https://hostingcanada.org/htpasswd-generator/) to generate the hash (use Bcrypt).
     - Or generate it with: `echo $(htpasswd -nB user1)`
5. Replace email for Let's encrypt (`mail@somedomain.com` in `configs/traefik.yml`)
    ```yaml
    certificatesResolvers:
      letsencrypt:
        acme:
          # ...
          email: mail@somedomain.com
    ```
6. Start container
   ```bash
   docker-compose up -d
   ```
7. Check that traefik is running smoothly
   ```bash
   docker-compose logs
   ```

### Traefik dashboard

The traefik dashboard is now available under:
```
https://reverse-proxy.somedomain.com
```
The dashboard shows you the configured routers, services, middleware, etc.

### Connect docker-compose service to reverse-proxy

```yaml
version: '3.7'
networks:
  frontend:
    external:
      name: reverse-proxy-docker-traefik_routing
services:
  someservice:
    restart: always
    # ...
    labels:
      - "traefik.enable=true"
      - "traefik.docker.network=reverse-proxy-docker-traefik_routing"
      # https
      - "traefik.http.routers.someservice.rule=Host(`someservice.com`)"
      - "traefik.http.routers.someservice.tls=true"
      - "traefik.http.routers.someservice.tls.certresolver=letsencrypt"
      - "traefik.http.routers.someservice.entrypoints=websecure"
      # http (redirect to https)
      - "traefik.http.routers.someservice-http.rule=Host(`someservice.com`)"
      - "traefik.http.routers.someservice-http.entrypoints=web"
      - "traefik.http.routers.someservice-http.middlewares=redirect-to-https@file"
    networks:
     - frontend
     - ...
```

**Password protection for service with basic auth**

```yaml
services:
  someservice:
    # ...
    labels:
      # ...
      - "traefik.http.routers.someservice.middlewares=someservice-auth"
      - "traefik.http.middlewares.someservice-auth.basicauth.users=user1:$2y$05$/x10KYbrHtswyR8POT.ny.H4fFd1n.0.IEiYiestWzE1QFkYIEI3m"
```

You can generate the **escaped** hash with the following command: `echo $(htpasswd -nB user1) | sed -e s/\\$/\\$\\$/g`
If you use a website like [this](https://hostingcanada.org/htpasswd-generator/) to generate the hash remember to escape the dollar signs (`$` -> `$$`) and use Bcrypt.

**Specifying port if service exposes multiple ports**

If your service exposes multiple ports Traefik does not know which one it should use.
With this line you can select one:

```yaml
services:
  someservice:
    # ...
    labels:
      # ...
      - "traefik.http.services.someservice.loadbalancer.server.port=8080"
```

## Setup for local development

### Setting up traefik

1. Clone repository
   ```bash
   git clone https://github.com/korridor/reverse-proxy-docker-traefik.git
   cd reverse-proxy-docker-traefik
   ```
2. Copy default config  
   ```bash
   ln -s docker-compose.local.yml docker-compose.yml
   ln -s configs-local configs
   ```
   
   If you want to change the configuration copy the configuration instead of creating a symlink.
   
   ```bash
   cp docker-compose.local.yml docker-compose.yml
   cp -r configs-local configs
   ```
3. If you want you can change the domain of the traefik dashboard (`reverse-proxy.test` in `configs/dynamic/dashboard.yml`)
   ```yaml
   http:
     routers:
       traefik:
         rule: Host(`reverse-proxy.test`)
         # ...
   ```
4. Start container
   ```bash
   docker-compose up -d
   ```
5. Check that traefik is running smoothly
   ```bash
   docker-compose logs
   ```

### Traefik dashboard

The traefik dashboard is now available under:
```
http://reverse-proxy.test
```
The dashboard shows you the configured routers, services, middlewares, etc.

### Connect docker-compose service to reverse-proxy

```yaml
version: '3.7'
networks:
  frontend:
    external:
      name: reverse-proxy-docker-traefik_routing
services:
  someservice:
    restart: always
    # ...
    labels:
      - "traefik.enable=true"
      - "traefik.docker.network=reverse-proxy-docker-traefik_routing"
      # http
      - "traefik.http.routers.someservice.rule=Host(`someservice.test`)"
      - "traefik.http.routers.someservice.entrypoints=web"
    networks:
     - frontend
     - ...
```

**Enabling service to send requests to itself (with someservice.test)**

```yaml
services:
  someservice:
    # ...
    extra_hosts:
      - "someservice.test:10.100.100.10"
```

**Specifying port if service exposes multiple ports**

If your service exposes multiple ports traefik does not know which one it should use.
With this config line you can select one:

```yaml
services:
  someservice:
    # ...
    labels:
      # ...
      - "traefik.http.services.someservice.loadbalancer.server.port=8080"
```

## Credits

I used the following resources to create this setup:

 - [Traefik docs](https://docs.traefik.io)
 - [Traefik v2 and Mastodon, a wonderful couple! by Nicolas Inden](https://www.innoq.com/en/blog/traefik-v2-and-mastodon/)
 - [GitHub repo traefik-example by jamct](https://github.com/jamct/traefik-example)

## License

This configuration is licensed under the MIT License (MIT). Please see [license file](license.md) for more information.

