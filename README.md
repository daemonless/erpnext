# :material-office-building: ERPNext

[![Build Status](https://img.shields.io/github/actions/workflow/status/daemonless/erpnext/build.yaml?style=flat-square&label=Build&color=green)](https://github.com/daemonless/erpnext/actions)
[![Last Commit](https://img.shields.io/github/last-commit/daemonless/erpnext?style=flat-square&label=Last+Commit&color=blue)](https://github.com/daemonless/erpnext/commits)

Open source ERP: accounting, inventory, manufacturing, CRM, HR and projects, built on the Frappe framework.

![ERPNext accounts dashboard](https://daemonless.io/images/screenshots/erpnext/image1f5eff.png)

| | |
|---|---|
| **Registry** | `ghcr.io/daemonless/erpnext` |
| **Source** | [https://github.com/frappe/erpnext](https://github.com/frappe/erpnext) |
| **Website** | [https://erpnext.com/](https://erpnext.com/) |

## Version Tags

| Tag | Description | Best For |
| :--- | :--- | :--- |
| `latest` | **Built from source**. Frappe and ERPNext `version-15`. | Most users. |

## Prerequisites

Before deploying, ensure your host environment is ready. See the [Quick Start Guide](https://daemonless.io/guides/quick-start) for host setup instructions.

ERPNext needs **MariaDB** and **Redis**; it will not start without them. Both are
in the stack below.

## Deploy

=== ":material-docker: Podman"

    === ":material-file-document-outline: Compose"

        **1.** Save as `.env`:

        ```env { data-zip-bundle="erpnext-podman" data-zip-filename=".env" }
        SITES_LOCATION=@CONTAINER_CONFIG_ROOT@/@ERPNEXT_SITES_PATH@
        DB_DATA_LOCATION=@CONTAINER_CONFIG_ROOT@/erpnext-mariadb
        REDIS_DATA_LOCATION=@CONTAINER_CONFIG_ROOT@/erpnext-redis
        SITE_NAME=erpnext.localhost
        DB_ROOT_PASSWORD=changeme
        ADMIN_PASSWORD=changeme
        ```

        **2.** Save as `compose.yaml`:

        ```yaml { data-zip-bundle="erpnext-podman" data-zip-filename="compose.yaml" }
        name: erpnext

        services:
          erpnext:
            image: ghcr.io/daemonless/erpnext:latest
            container_name: erpnext
            network_mode: host
            # always (not unless-stopped) so FreeBSD's podman rc.d auto-starts it at boot
            restart: always
            environment:
              PUID: "@PUID@"
              PGID: "@PGID@"
              TZ: "@TZ@"
              SITE_NAME: ${SITE_NAME}
              ADMIN_PASSWORD: ${ADMIN_PASSWORD}
              DB_HOST: 127.0.0.1
              DB_PORT: "3306"
              DB_ROOT_PASSWORD: ${DB_ROOT_PASSWORD}
              REDIS_CACHE: redis://127.0.0.1:6379/0
              REDIS_QUEUE: redis://127.0.0.1:6379/1
            volumes:
              - ${SITES_LOCATION}:/app/frappe-bench/sites
            depends_on:
              - mariadb
              - redis

          mariadb:
            image: ghcr.io/daemonless/mariadb:11.4
            container_name: erpnext-mariadb
            network_mode: host
            restart: always
            environment:
              MYSQL_ROOT_PASSWORD: ${DB_ROOT_PASSWORD}
              TZ: "@TZ@"
            volumes:
              - ${DB_DATA_LOCATION}:/config

          redis:
            image: ghcr.io/daemonless/redis:latest
            container_name: erpnext-redis
            network_mode: host
            restart: always
            environment:
              TZ: "@TZ@"
            volumes:
              - ${REDIS_DATA_LOCATION}:/config
        ```

        **3.** Deploy:

        ```bash
        mkdir -p @CONTAINER_CONFIG_ROOT@/@ERPNEXT_SITES_PATH@ @CONTAINER_CONFIG_ROOT@/erpnext-mariadb @CONTAINER_CONFIG_ROOT@/erpnext-redis
        chown -R @PUID@:@PGID@ @CONTAINER_CONFIG_ROOT@/@ERPNEXT_SITES_PATH@
        podman-compose up -d
        ```

=== ":appjail-appjail: AppJail"

    !!! warning
        Exposing ports in AppJail means that your service can be reached from remote hosts. If that is not your intention, do not expose the ports and communicate with the service using the jail's IPv4 address or hostname assigned by the virtual network.

    === ":material-file-document-outline: Director"

        **.env**:

        ```
        # .env

        DIRECTOR_PROJECT=erpnext
        PUID=1000
        PGID=1000
        TZ=UTC
        SITE_NAME=erpnext.localhost
        ADMIN_PASSWORD=changeme
        DB_HOST=127.0.0.1
        DB_PORT=3306
        DB_ROOT_PASSWORD=changeme
        REDIS_CACHE=redis://127.0.0.1:6379/0
        REDIS_QUEUE=redis://127.0.0.1:6379/1
        DB_ROOT_USER=
        SOCKETIO_PORT=
        GUNICORN_WORKERS=
        GUNICORN_THREADS=
        GUNICORN_TIMEOUT=
        DEP_WAIT_TIMEOUT=
        ```

        **appjail-director.yml**:

        ```yaml
        # appjail-director.yml

        options:
          - alias:
          - ip4_inherit:
        services:
          erpnext:
            name: erpnext
            options:
              - container: 'args:--pull'
              - expose: ':8080 proto:tcp'
            oci:
              user: root
              environment:
                - PUID: !ENV '${PUID}'
                - PGID: !ENV '${PGID}'
                - TZ: !ENV '${TZ}'
                - SITE_NAME: !ENV '${SITE_NAME}'
                - ADMIN_PASSWORD: !ENV '${ADMIN_PASSWORD}'
                - DB_HOST: !ENV '${DB_HOST}'
                - DB_PORT: !ENV '${DB_PORT}'
                - DB_ROOT_PASSWORD: !ENV '${DB_ROOT_PASSWORD}'
                - REDIS_CACHE: !ENV '${REDIS_CACHE}'
                - REDIS_QUEUE: !ENV '${REDIS_QUEUE}'
                - DB_ROOT_USER: !ENV '${DB_ROOT_USER}'
                - SOCKETIO_PORT: !ENV '${SOCKETIO_PORT}'
                - GUNICORN_WORKERS: !ENV '${GUNICORN_WORKERS}'
                - GUNICORN_THREADS: !ENV '${GUNICORN_THREADS}'
                - GUNICORN_TIMEOUT: !ENV '${GUNICORN_TIMEOUT}'
                - DEP_WAIT_TIMEOUT: !ENV '${DEP_WAIT_TIMEOUT}'
            volumes:
              - erpnext_sites: /app/frappe-bench/sites
          erpnext-mariadb:
            name: erpnext_mariadb
            options:
              - from: ghcr.io/daemonless/mariadb:11.4
              - template: !ENV '${PWD}/template.conf'
            volumes:
              - mariadb_data: /config
          erpnext-redis:
            name: erpnext_redis
            options:
              - from: ghcr.io/daemonless/redis:latest
              - template: !ENV '${PWD}/template.conf'
            volumes:
              - redis_data: /config
        volumes:
          erpnext_sites:
            device: '/erpnext/sites'
          mariadb_data:
            device: !ENV '${MARIADB_DATA_LOCATION}'
          redis_data:
            device: !ENV '${REDIS_DATA_LOCATION}'
        ```

        **Makejail**:

        ```
        # Makejail

        ARG tag=latest

        OPTION container=boot
        OPTION overwrite=force
        OPTION from=ghcr.io/daemonless/erpnext:${tag}
        ```

        Save the files above, then run `appjail-director up`.

Access ERPNext at **http://your-host:8080** and log in as
**Administrator** with the `ADMIN_PASSWORD` you set.

### Interactive Configuration

<div class="placeholder-settings-panel"></div>

## First run

Creating the site takes several minutes — ERPNext installs its full schema and
then works through a large batch of background jobs. The web UI answers before
that finishes; if you land on *"Setting up your system"*, let it run and reload.

`ADMIN_PASSWORD` and `DB_ROOT_PASSWORD` are read **only** when the site is
created. Every later start runs `bench migrate` against the existing site
instead, so changing them afterwards has no effect — change the Administrator
password from the UI.

`MYSQL_ROOT_PASSWORD` and `DB_ROOT_PASSWORD` must match: ERPNext uses the latter
to create its database and user on first run.

## Environment Variables

| Variable | Description |
|----------|-------------|
| `SITE_NAME` | Frappe site to create and serve. Any hostname reaches it, so it need not resolve in DNS. |
| `ADMIN_PASSWORD` | Password for the `Administrator` account. First run only. |
| `DB_ROOT_PASSWORD` | MariaDB root password, used once to create the site's database and user. First run only. |
| `DB_HOST` / `DB_PORT` | MariaDB address (`127.0.0.1:3306` with host networking) |
| `REDIS_CACHE` | Redis URL for the cache |
| `REDIS_QUEUE` | Redis URL for the job queue and realtime |
| `GUNICORN_WORKERS` | Web worker processes (default `2`) |
| `GUNICORN_THREADS` | Threads per web worker (default `4`) |
| `GUNICORN_TIMEOUT` | Seconds before a request is killed (default `120`); raise for long reports |
| `DEP_WAIT_TIMEOUT` | Seconds to wait for MariaDB and Redis before giving up (default `180`) |

## FreeBSD Notes

### Hostnames

Frappe picks the site from the `X-Frappe-Site-Name` header rather than the URL,
and this image sends `SITE_NAME` on every request. Any address that reaches the
container works — `http://server.lan:8080`, an IP, or a reverse proxy
— without `SITE_NAME` resolving in DNS.

### MariaDB version

Use **MariaDB 11.4** — the version this image is tested against. Frappe v15
prints a warning for any server above 10.8, but it is only a warning, and it
covers every MariaDB the registry ships (10.11 included).

Frappe creates its own database as `utf8mb4`, so the server's default character
set does not matter. PostgreSQL is not supported — ERPNext itself only targets
MariaDB.

### DuckDB sync unavailable

The **DuckDB sync** doctype does not work. Frappe pins `pyarrow~=25.0.0`, whose
bindings will not compile against the Arrow C++ 24 that FreeBSD ports carries.
Everything else — accounting, stock, manufacturing, CRM, HR, projects and print
formats — is unaffected.

### Network mode

The stack uses `network_mode: host`, so all three services talk over
`127.0.0.1`. Only port `8080` needs to be reachable externally.

## Management

```bash
# View logs
podman-compose logs -f
podman logs -f erpnext

# Backup (archives land in the sites volume, under <site>/private/backups)
podman exec erpnext bench --site ${SITE_NAME} backup --with-files

# Restart / update
podman-compose restart
podman-compose pull && podman-compose up -d
```