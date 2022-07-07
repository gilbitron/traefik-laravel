# Traefik Laravel

This is a simple [Traefik proxy](https://traefik.io/traefik/) setup for running multiple [Laravel Sail](https://laravel.com/docs/9.x/sail) instances. It uses the [bjornsnoen/minica-api](https://github.com/bjornsnoen/minica-api) package to auto-generate self-signed SSL certificates.

## Install

First, create a new network called `web`:

```
docker network create web
```

Then, simply run the docker compose file:

```
docker composer up -d
```

You should now be able to visit [https://traefik.test](https://traefik.test) and [https://whoami.test](https://whoami.test) to confirm that the proxy is working properly.

## Adding Sail Apps

Traefik proxy auto-discovery works by adding labels to your containers. To have a Laravel Sail app discovered by Traefik, update the `docker-compose.yml` to look like this:

```diff
version: '3'
services:
  laravel:
    build:
      context: ./vendor/laravel/sail/runtimes/8.1
      dockerfile: Dockerfile
      args:
        WWWGROUP: '${WWWGROUP}'
    image: sail-8.1/app
    extra_hosts:
      - 'host.docker.internal:host-gateway'
    ports:
-     - '${APP_PORT:-80}:80'
      - '${VITE_PORT:-5173}:${VITE_PORT:-5173}'
    environment:
      WWWUSER: '${WWWUSER}'
      LARAVEL_SAIL: 1
      XDEBUG_MODE: '${SAIL_XDEBUG_MODE:-off}'
      XDEBUG_CONFIG: '${SAIL_XDEBUG_CONFIG:-client_host=host.docker.internal}'
    volumes:
      - '.:/var/www/html'
    networks:
      - sail
+     - web
+   labels:
+     - "traefik.enable=true"
+     - "traefik.http.routers.laravel.rule=Host(`laravel.test`)"
+     - "traefik.http.routers.laravel.tls=true"
+     - "traefik.http.services.laravel.loadbalancer.server.port=80"
+     - "traefik.docker.network=web"

# ...

networks:
  sail:
    driver: bridge
+ web:
+   external: true
```

Note, we've had to rename the container to "laravel" as we can't use "laravel.test" in the traefik labels. You should also add the following to your `.env`:

```
APP_SERVICE=laravel
```

After running `sail up` your site should now be running at [https://laravel.test](https://laravel.test).

## Trusting Certificates

You can add `./certificates/minica.pem` file to your keychain or local trust store to trust the certificates.
