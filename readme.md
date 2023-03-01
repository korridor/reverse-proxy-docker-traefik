# Traefik 2 config


This is a resuseable traefik config for usage on a virtual server or for local debelopment using docker-compose.   
It uses:
 - Traefik 2
 - docker-compose
 - Let's encrypt

## Table of content

- [Traefik 2 config](#traefik-2-config)
  - [Table of content](#table-of-content)
  - [Production setup](#production-setup)
    - [Setting up traefik](#setting-up-traefik)
    - [Traefik dashboard](#traefik-dashboard)
    - [Connect docker-compose service to reverse-proxy](#connect-docker-compose-service-to-reverse-proxy)
    - [SSL configuration](#ssl-configuration)
    - [Global middlewares](#global-middlewares)
    - [Access Logs](#access-logs)
  - [Setup for local development](#setup-for-local-development)
    - [Setting up traefik](#setting-up-traefik-1)
    - [Traefik dashboard](#traefik-dashboard-1)
    - [Connect docker-compose service to reverse-proxy](#connect-docker-compose-service-to-reverse-proxy-1)
    - [Enable SSL locally](#enable-ssl-locally)
    - [Enable SSL in the docker-compose file](#enable-ssl-in-the-docker-compose-file)
  - [Credits](#credits)
  - [License](#license)

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
version: '3.8'
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

### SSL configuration

Per default the SSL configuration is set so that [SSL Labs](https://www.ssllabs.com/) gives an `A` rating.

If you want an `A+` rating, you need to use HSTS (HTTP Strict Transport Security).
The setup includes a global middleware called `hsts-minimal@file` that can be used to activate HSTS in a simple setting.
See "Global middlewares" for more information.

### Global middlewares

**hsts-minimal@file**

Adds the HSTS header to the HTTP response without `includeSubDomains` and `preload`.
The `max-age` is set to one year / 31536000 seconds.

**hsts-standard@file**

Adds the HSTS header to the HTTP response with `includeSubDomains` and no `preload`.
The `max-age` is set to one year / 31536000 seconds.

**hsts-full@file**

Adds the HSTS header to the HTTP response with `includeSubDomains` and `preload`.
The `max-age` is set to one year / 31536000 seconds.

**redirect-to-https@file**

Adds a permanent redirect to HTTPS.

**redirect-non-www-to-www@file**

Adds a permanent redirect (HTTP 301) from non-www domains to the HTTPS www domain
Examples:
- `https://example.test` -> `https://www.example.test`
- `http://example.test` -> `https://www.example.test`

**redirect-www-to-non-www@file**

Adds a permanent redirect (HTTP 301) from www domains to the HTTPS non-www domain
Examples:
- `https://www.example.test` -> `https://example.test`
- `http://www.example.test` -> `https://example.test`

### Access Logs

To enable the traefik access logs in the production configuration, open the file `traefik.yml` within the config folder and uncomment the `accessLog` section.

```yml
# Access logs
accessLog: {}
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
version: '3.8'
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

### Enable SSL locally

1. Install [mkcert](https://github.com/FiloSottile/mkcert)

For example on macOS:

```bash
brew install mkcert
brew install nss # if you use Firefox
```

Now install the local CA:

```bash
mkcert -install
```

3. Generate certificate

Replace `someservice` with the domains that you are using for local development.

```bash
cd certificates
mkcert -key-file local.key.pem -cert-file local.cert.pem "*.local" "*.test" "*.someservice.test" "*.someservice.local"
```

### Enable SSL in the docker-compose file

```yaml
version: '3.8'
networks:
  frontend:
    external:
      name: reverse-proxy-docker-traefik_routing
services:
  someservice:
    restart: always
    # ...
    labels:
      - ...
      # http
      - ...
      # https
      - "traefik.http.routers.someservice-https.rule=Host(`someservice.test`)"
      - "traefik.http.routers.someservice-https.entrypoints=websecure"
      - "traefik.http.routers.someservice-https.tls=true"
    networks:
     - frontend
     - ...
```

## Credits

I used the following resources to create this setup:

 - [Traefik docs](https://docs.traefik.io)
 - [Traefik v2 and Mastodon, a wonderful couple! by Nicolas Inden](https://www.innoq.com/en/blog/traefik-v2-and-mastodon/)
 - [GitHub repo traefik-example by jamct](https://github.com/jamct/traefik-example)

## License

This configuration is licensed under the MIT License (MIT). Please see [license file](license.md) for more information.

