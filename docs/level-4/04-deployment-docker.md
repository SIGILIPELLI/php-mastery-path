# 04 · Deployment (Docker for PHP-FPM + Nginx)

Every module so far ran PHP directly from the command line. Production PHP
web applications usually run behind **PHP-FPM** (a process manager that
handles PHP execution) fronted by **Nginx** (a web server that serves
static files directly and proxies dynamic requests to PHP-FPM) — and
**Docker** packages that whole setup so it runs identically on a laptop, a
CI runner, and a production host. This module isn't run against a live
Docker daemon (not available in this environment), but every file below is
a real, deployable configuration — described accurately rather than
demonstrated live.

## Why PHP-FPM + Nginx, not `php -S`

The `php -S localhost:8000` built-in server used for quick local testing is
single-threaded and explicitly documented by PHP itself as unfit for
production — it can't handle concurrent requests well and has no process
supervision. PHP-FPM solves this: it maintains a **pool of worker
processes**, each handling one request at a time, with the pool size tuned
to the server's resources. Nginx sits in front, serving `.css`/`.js`/images
directly (fast, no PHP involved) and forwarding only requests that need
PHP execution to the FPM pool over a Unix socket or TCP port.

## The Dockerfile

```dockerfile
# Dockerfile
FROM php:8.3-fpm-alpine

# System dependencies + PHP extensions the app needs
RUN apk add --no-cache sqlite-dev \
    && docker-php-ext-install pdo pdo_sqlite opcache

# Production OPcache tuning -- see Level 3's Performance & Caching module
RUN { \
        echo 'opcache.enable=1'; \
        echo 'opcache.validate_timestamps=0'; \
        echo 'opcache.memory_consumption=128'; \
    } > /usr/local/etc/php/conf.d/opcache-prod.ini

WORKDIR /var/www/html

# Install dependencies BEFORE copying app code -- Docker layer caching means
# this expensive step only reruns when composer.json/.lock actually change,
# not on every source code edit.
COPY composer.json composer.lock ./
RUN composer install --no-dev --optimize-autoloader --no-interaction

COPY . .

RUN chown -R www-data:www-data /var/www/html

USER www-data
EXPOSE 9000
CMD ["php-fpm"]
```

`opcache.validate_timestamps=0` is a production-only setting: OPcache
normally checks each file's modification time on every request to detect
source changes, which is convenient in development but pure overhead in
production, where the deployed code never changes without a fresh
container build. Disabling that check means a `composer install` or file
edit inside a *running* container's filesystem won't be picked up — which
is exactly the point: production deploys happen by building and replacing
the whole container, not by patching a running one.

## The Nginx configuration

```nginx
# nginx.conf
server {
    listen 80;
    server_name _;
    root /var/www/html/public;
    index index.php;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        fastcgi_pass app:9000;              # "app" = the PHP-FPM container's service name
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }

    location ~ /\.(?!well-known) {
        deny all;                            # block access to .env, .git, etc.
    }
}
```

`fastcgi_pass app:9000` names the PHP-FPM *service*, not `localhost` — this
only resolves correctly inside Docker's internal network, which is what
`docker-compose.yml` sets up next. The `deny all` block on dotfiles matters
more than it looks: without it, a misconfigured server would happily serve
`/.env` as a plain-text file to anyone who requests it by name.

## Wiring both together with Docker Compose

```yaml
# docker-compose.yml
services:
  app:
    build: .
    volumes:
      - ./storage:/var/www/html/storage   # persists SQLite file / logs across restarts
    environment:
      - APP_ENV=production

  web:
    image: nginx:alpine
    ports:
      - "8080:80"
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf:ro
      - ./public:/var/www/html/public:ro
    depends_on:
      - app
```

```bash
docker compose up --build
# Nginx listens on host port 8080, proxying PHP requests to the "app" service's FPM pool
```

Two services, one network Docker Compose creates automatically — `web` can
reach `app` by its service name (`app:9000`) exactly as the Nginx config
above expects, with no manual network configuration.

## PHP traps

**Copying application code before running `composer install` busts the
Docker build cache on every single source change**, forcing a full
dependency reinstall even when `composer.json` didn't change. The
Dockerfile above deliberately copies `composer.json`/`composer.lock`
*first*, installs, and only *then* copies the rest — Docker's layer cache
treats each `COPY`/`RUN` as a step that's skipped and reused if its inputs
are unchanged, so editing `src/TaskController.php` alone re-triggers only
the final `COPY . .` layer, not the (slow) dependency install.

**`opcache.validate_timestamps=0` in a *development* container makes code
edits appear to do nothing** — a very confusing debugging session for
anyone who forgot which environment they're in. Keep the aggressive OPcache
settings in a production-only config file, and use the framework-default
(timestamp-validating) settings for any container meant for local
iteration.

**Running the container process as root is a real security gap, not just
style.** A PHP-FPM process compromised through an application
vulnerability (a file upload flaw, a deserialization bug) inherits whatever
privileges the process runs with — `USER www-data` in the Dockerfile above
limits the blast radius to what that unprivileged user can touch, instead
of full filesystem access inside the container.

## Deployment cheat sheet

| Piece | Role |
|---|---|
| PHP-FPM | Process pool that actually executes PHP for each request |
| Nginx | Serves static assets directly; proxies `.php` requests to FPM |
| `fastcgi_pass app:9000` | Nginx forwards dynamic requests to the FPM service |
| Dockerfile layer ordering | Dependencies installed before app code copied — cache-friendly |
| `opcache.validate_timestamps=0` | Production-only: skip per-request file change checks |
| `USER www-data` | Run the PHP process as an unprivileged user, not root |
| `docker compose up --build` | Builds and starts both services on one shared network |

## Stretch goals

*(This module has no runnable PHP demo since it's infrastructure
configuration — practice the concepts as stretch goals instead of an
exercise.)*

- If Docker is available on your machine, containerize the
  [REST API project](../level-3/10-project-rest-api.md) using the
  Dockerfile pattern above (swap `pdo_sqlite` for the extension it already
  needs), add a `public/index.php` front controller that dispatches through
  its `Router`, and confirm `curl http://localhost:8080/tasks` returns JSON
  from a running container.
- Add a multi-stage build: a `composer` stage using the official
  `composer:2` image to run `composer install`, copying only
  `vendor/` into the final `php:8.3-fpm-alpine` stage — this keeps
  Composer itself (and any dev tools it might pull in) out of the final
  production image entirely.
- Add a `HEALTHCHECK` instruction to the Dockerfile that curls a
  `/health` endpoint your app exposes, and explain in a comment why a
  container orchestrator (Docker Swarm, Kubernetes) needs this to safely
  replace unhealthy containers during a rolling deploy.
