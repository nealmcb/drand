# Drand on docker

The main readme for the drand project is [here](./README.md). This readme describes how to run `drand` on `docker` and `docker-compose`.

## Prerequisites for this guide

a VPS with the following software setup:

1. `docker >= 17.12`
2. `docker-compose >= 1.18`

on your host:

1. `go >= 1.12`

## Generate keys

Before creating the docker, let's generate keys locally (on the host)

```
go get -u github.com/dedis/drand
drand generate-keypair <address>
```

Where `<address>` is for instance `drand.yourserver.com:1234` or `yourserver.com:1234`.

This generates keys in `~/.drand/keys/` on the host.

## Docker & Docker-compose setup

Create a `Dockerfile` with the following contents:

```
FROM golang:1.12.5-alpine # update this number when possible, but keep "alpine" for a lightweight image
RUN apk update
RUN apk upgrade
RUN apk add git
RUN go get -u github.com/dedis/drand
WORKDIR /
ENTRYPOINT ["drand", "--verbose", "2", "start", "--listen", "0.0.0.0:8080"]
```

**Note:** if you intend to use `drand` behind a reverse-proxy, add `--tls-disable` here; please read "Docker behind reverse proxy setup" at the end of this guide. 

Let's continue with docker-compose. It will help keeping our container running, plus manage volumes, etc.

Create the following `docker-compose.yml` :

```
drand:
  build: .
  volumes:
    - ./data:/root/.drand
  ports:
    - "0.0.0.0:1234:8080"
```

Finally, also create a `data` folder to hold the keys and settings. It **HAS** to have permissions `740`.

```
mkdir data
chmod 740 data
```

Drand can then be built into an image with

```
docker-compose up --build
```

Finally, the image is ready for DKG. Just start it and let it run in background with 

```
docker-compose up -d
```

One can also set a restart policy in the `docker-compose.yml` by adding `restart: always`.


## Distributed Key Generation

If you did the setup above, you have a container running, loaded with your keys. It still misses two things:

1. the group.toml file corresponding to other participants. For this, you have to exchange keys manually, e.g., via email.
2. running the DKG protocol to bootstrap drand.

Fortunately, with our docker-compose volumes, it's now very easy to add things into the running container. Just add your `group.toml` into the root of the `data` folder (**NOT** in the `data/groups/` folder; this one is manually managed by drand, don't touch it).

Then, open a CLI into your running docker.

First find its id on the host:

```
$ docker ps
697e4766f8b2        drand_drand             "drand --verbose 2 s…"   11 minutes ago      Up 9 minutes
```

The id of the container is `697e4766f8b2`. Enter it by running:

```
docker exec -it 697e4766f8b2 /bin/sh
```

Then, you're inside the container; tell drand to run the DKG like so:

```
drand share /root/.drand/group.toml
```

**Notice the full path** `/root/.drand/group.toml` and not `group.toml` nor `./group.toml`

## Checking the logs

From outside the container, run `docker-compose logs`

## Updating drand

To update the docker, simply shut it down

```
docker-compose down
```

and to fully rebuild it, you need to first clean the already-used layers (for some reason Docker is confused and thinks nothing has changed, and it keeps rebuilding the old version). Caution! this delete *all* your *unused* containers and networks; it's typically fine, but just be aware of it.

```
docker system prune -a
```

Then rebuild and restart it

```
docker-compose up --build -d
```

## Reset the docker state (without losing the keys)

This part is if you need to reset drand's internal state without loosing the keys. 

### Method 1: using `drand clean`

First, try using this method. If that doesn't work, use the method below.

Find drand's container id on the host, and enter it:

```
$ docker ps
697e4766f8b2        drand_drand             "drand --verbose 2 s…"   11 minutes ago      Up 9 minutes

$ docker exec -it 697e4766f8b2 /bin/sh
```

Then simply call:

```
drand reset
```

Exit the container with `CTRL-C`. Then, on the host, I advise you to restart the container (to make sure the drand deamon has a clean restart and can reload its cleaned config):

```
docker-compose down
docker-compose up --build -d
```

### Method 2: doing a reset manually

The method above relies on `drand clean`, which could theoretically fail. If you want a manual hard-reset, start by killing the container:

```
docker-compose down
```

Delete what you want to reset in `data`: typically, you absolutely want to **keep** `data/keys`, **especially** if you shared those keys to create a `group.toml` with other people. For instance if the DKG failed, remove `data/db` and `data/groups`. Notice that if you added the `group.toml` into the root of `data` as suggested, it should still be there (don't delete it unless you want to change the group).

Then, rebuild the image from scratch:

```
docker system prune -a
docker-compose up --build -d
```

Check that things are running with

```
docker-compose logs
```

You're now back to the step "Distributed Key Generation" of this guide.

## Docker behind reverse proxy setup

Typically, the TLS part of my VPS is managed by a single reverse proxy, which then proxies multiple services running locally with docker.

There is one subtletly: you need to forward _both_ GRPC (used by drand "core") and web traffic (for the web interface). To forward GRPC, you need to have nginx `1.13.10` or above, it's a fairly recent addition.

Then, you need to forward differently traffic to `/` and to `/api/`. Here's an example configuration for `nginx`:

```
server {
  server_name drand.lbarman.ch;
  listen 443 ssl http2;
  ssl_protocols   SSLv3 TLSv1 TLSv1.1 TLSv1.2;
  ssl_ciphers   HIGH:!aNULL:!MD5;
  location / {
    grpc_pass grpc://localhost:1234;
  }
  location /api/ {
    proxy_pass http://localhost:1234; 
    proxy_set_header Host $host;
  }
  
  ssl_certificate /etc/letsencrypt/live/.../fullchain.pem; # managed by Certbot
  ssl_certificate_key /etc/letsencrypt/live/.../privkey.pem; # managed by Certbot
}
```

Naturally, the local `drand` in the container does not need to worry about TLS anymore. Add the `--tls-disable` flag in the `Dockerfile`:

```
ENTRYPOINT ["drand", "--verbose", "2", "start", "--tls-disable", "--listen", "0.0.0.0:8080"]
```

*But* notice that to others, you'll still be using TLS (handled by your reverse proxy), so make sure you generate keys using an https address, and the flag `TLS=true`.