layout: page
title: "My postgres image adventure"
permalink: /posts/my-postgres-image-adventure

## TL;DR

If you think your postgres container is not behaving as it should, check if there's any ghost running with `lsof` and kill if any

If you are interested in my story, please read on.

So it was a fine Friday evening, what could happen in this beautiful day of week right?

It started when I tried to run an API in my local for the first time. This should be nothing more than bringing up the DB via `docker-compose up -d` with the following yaml and a `go run`.

```yaml
version: '3.7'
services:
  database:
    container_name: db-${CONTAINER_SUFFIX:-local}
    image: postgres:13-alpine
    ports:
      - "5432:5432"
    restart: always
    healthcheck:
      test: [ "CMD-SHELL", "pg_isready" ]
      interval: 30s
      timeout: 30s
      retries: 3
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password
      POSTGRES_DB: test
```

However, `go run` gave me

```
pq: role "user" does not exist
```

What? Which part of `POSTGRES_USER: user` did you not understand?

The first thing I tried was to change the .env to point to a deployed DB and `go run`, to make sure the app runs. It worked.

Ok, the code was fine, something was wrong with the DB. I then tried `psql -h localhost -U user -p 5432` and it gave me the same `pq: role "user" does not exist` error as expected.

Naturally I `docker ps`ed and saw

```
CONTAINER ID   IMAGE                COMMAND                  CREATED          STATUS                             PORTS                                       NAMES
bddd6e056a10   postgres:13-alpine   "docker-entrypoint.s…"   25 seconds ago   Up 24 seconds (health: starting)   0.0.0.0:5432->5432/tcp, :::5432->5432/tcp   db-local
```

so I jumped into the container by `docker exec -it bddd6e056a10 /bin/sh`, and then ran `psql -h localhost -U user -p 5432` there, it worked! All `\du`, `\l`, `\dt` were working as expected.

Wow. Port forwarding working, everything running fine in the container, but I couldn't reach it. My good friend Google didn't seem to be helping for a few hours.

Out of frustration, I `docker-compose down`ed and tried `psql -h localhost -U user -p 5432`, and the same `pq: role "user" does not exist`.

Yeah, that's enough of a night... Wait, what? My box was torn, shouldn't you tell me "Connection refused"?

Aha! You ghost! I immediately `lsof -nP -i:5432`ed and got

```
COMMAND   ... NODE NAME
postgres  ... TCP  [::1]:5432 (LISTEN)
postgres  ... TCP  127.0.0.1:5432 (LISTEN)
```

Oh...

```
➜  ~ pg_ctl -D /usr/local/var/postgres status
pg_ctl: server is running (PID: 10843)
```

Problem found, I somehow started a local postgres service inadvertently.

All I had to do was `brew services stop postgresql`, then `docker-compose up -d`, and `go run`.

But why did `docker-compose up -d` work while the port is being used? To be continued...
