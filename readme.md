# Traefik 2 config

This is a resuseable traefik config for usage on a vServer using docker-compose.
It uses:
 - Traefik 2.1.6
 - docker-compose
 - Let's encrypt

## Setup production

1. Clone repository
2. Copy default config  
   ```bash
   cp docker-compose.prod.yml docker-compose.yml
   cp configs-prod configs
   echo "{}" > certificates/acme.json
   chmod 600 certificates/acme.json
   ```
3. Replace domain for dashboard (`reverse-proxy.somedomain.com` in `config/dynamic/dashboard.yml`)
4. Replace password for admin account (in `config/dynamic/dashboard.yml`)   
    You can use a website like [this](https://hostingcanada.org/htpasswd-generator/) to generate the hash. Do not forget to escape dollar signs by doubling them. (`$` -> `$$`)
5. Replace email for Let's encrypt (`mail@somedomain.com` in `config/traefik.yml`)

### Connect docker-compose service to reverse-proxy

```yaml
version: '3.7'
networks:
  frontend:
    external:
      name: traefik_routing
services:
  someservice:
    # ...
    labels:
      traefik.enable: "true"
      traefik.docker.network: "traefik_routing"
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

