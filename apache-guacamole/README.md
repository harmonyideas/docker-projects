# Apache Guacamole (Docker Compose)

Apache Guacamole is a clientless remote desktop gateway. Users connect to their
desktops (VNC, RDP, SSH) from a web browser without installing client software.

This folder contains a Docker Compose stack with these services:

| Service    | Image                  | Role                                              |
|------------|------------------------|---------------------------------------------------|
| guacd      | guacamole/guacd:1.5.5  | Protocol proxy used by the Guacamole web app      |
| postgres   | postgres:16            | Stores Guacamole users, connections and settings  |
| guacamole  | built from `build: .`  | Guacamole web application (needs a Dockerfile)    |
| nginx      | nginx (unpinned)       | Optional TLS front end, published on port 8443    |

Guacamole is served at `/guacamole` on the Guacamole port 8080 inside the Docker
network. Through nginx it is available at `https://<host>:8443/guacamole/`.

## Setup

    cp .env.example .env

Then edit `.env` and set `POSTGRES_PASSWORD` to a long random value. The compose
file reads it with `${POSTGRES_PASSWORD:?...}` and refuses to start without it.
`.env` is git-ignored. Never commit it.

| Variable           | Used by            | Purpose                              |
|--------------------|--------------------|--------------------------------------|
| POSTGRES_PASSWORD  | postgres, guacamole | Password for the `guacamole_user` database user |

The upstream default Guacamole login is `guacadmin` / `guacadmin`. It exists only
if the database was initialised from the upstream `initdb.sql`. Change it immediately after the first login.

## Upgrade note (PostgreSQL 13 to 16)

The postgres image was changed from `postgres:13.4-buster` (end of life) to
`postgres:16`. PostgreSQL data files are not compatible across major versions.
If `./data` already holds a PostgreSQL 13 database, dump it with the old image,
move `./data` aside, start the new stack and restore the dump. Do not start
16 on top of an existing 13 data directory.

## Missing files

The compose file references files and folders that are NOT in this repository.
The stack will not start until they exist. Nothing below has been generated.
Each item is a human decision or a manual step.

1. Dockerfile (for the `guacamole` service, `build: .`)
   Must exist in this folder. Either write a Dockerfile that builds the
   Guacamole web app, or replace `build: .` with `image: guacamole/guacamole`.

2. `./init/initdb.sql` (mounted into the postgres container)
   Generate once:

       mkdir -p init
       docker run --rm guacamole/guacamole /opt/guacamole/bin/initdb.sh --postgres > ./init/initdb.sql
       chmod +x ./init

   Postgres ignores the script if the folder is not executable. The script runs only
   when the database directory is empty.

3. `./nginx/templates/guacamole.conf.template` (mounted into the nginx container)
   An nginx server block that listens on 443 with SSL, uses
   `/etc/nginx/ssl/self.cert` and `/etc/nginx/ssl/self-ssl.key`, and proxies to
   `http://guacamole:8080/guacamole/` with WebSocket upgrade headers.

4. `./nginx/ssl/self.cert` and `./nginx/ssl/self-ssl.key` (TLS certificate and key)
   Create a self-signed pair or use your own certificate, with these exact file names.

5. `prepare.sh` and `reset.sh` are referenced in the compose header comments.
   Both are missing. Docker Compose does not require them.

## Ports

nginx publishes 8443 on all interfaces. The guacamole service exposes 8080 only
inside the Docker network. To use your own reverse proxy instead of nginx, remove
the nginx service and change the guacamole port to `8080:8080/tcp`. Guacamole is
then on `:8080/guacamole`, not `/`.
