# Traefik 2 config

This is a resuseable traefik config for usage on a vServer using docker-compose.
It uses:
 - Traefik 2.1
 - docker-compose
 - Let's encrypt

## Setup production

1. Clone repository
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
    # ...
    labels:
      traefik.enable: "true"
      traefik.docker.network: "reverse-proxy-docker-traefik_routing"
      # https
      traefik.http.routers.someservice.rule: "Host(`someservice.com`)"
      traefik.http.routers.someservice.tls: "true"
      traefik.http.routers.someservice.tls.certresolver: "letsencrypt"
      traefik.http.routers.someservice.entrypoints: "websecure"
      # http (redirect to https)
      traefik.http.routers.someservice-http.rule: "Host(`someservice.com`)"
      traefik.http.routers.someservice-http.entrypoints: "web"
      traefik.http.routers.someservice-http.middlewares: "redirect-to-https@file"
    networks:
     - frontend
     - ...
```

## Setup for local development

Special config for local development is coming soon.
For now the production setup can be used.

## Credits

I used the following resources to create this setup:

 - [Traefik docs](https://docs.traefik.io)
 - [Traefik v2 and Mastodon, a wonderful couple! by Nicolas Inden](https://www.innoq.com/en/blog/traefik-v2-and-mastodon/)
 - [GitHub repo traefik-example by jamct](https://github.com/jamct/traefik-example)

## License

This package is licensed under the MIT License (MIT). Please see [license file](license.md) for more information.

