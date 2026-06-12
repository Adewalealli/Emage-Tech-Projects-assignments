# Assignment 04 — Adewale Alli

**GitHub username:** adewalealli
**Date completed:** 2026-06-11

## 1. Answers to the 11 questions

**Q1 — default-bridge DNS gap:**
On the default bridge, the API container had DNS configuration, but the name `db` did not resolve. My `/etc/resolv.conf` showed a nameserver, so the problem was not that DNS was completely missing. The problem was that Docker's default bridge does not provide automatic container-name DNS records the way a user-defined bridge network does. `dig +short db` returned no IP, and `getent hosts db` returned `not found`, so `DB_HOST=db` could not work on the default bridge.

**Q2 — db IP after restart + why hard-coding is wrong:**
In my run, the db IP stayed the same after restart: `172.17.0.3`. However, hard-coding that IP is still not reliable because the IP belongs to a container instance, not to the database service as a stable identity. If the container is removed and recreated, or if the network changes, the IP can change. In production, hard-coding an IP and lacking DNS are the same class of bug because the application depends on unstable infrastructure instead of a stable service name.

**Q3 — subnet/gateway + flags to control them:**
Docker picked subnet `172.20.0.0/16` and gateway `172.20.0.1` for my manually created `cohort-net`. To control these values, I would create the network with flags like `--subnet 172.30.0.0/24` and `--gateway 172.30.0.1`. This would be useful if Docker's automatic subnet conflicted with a VPN, office network, or another local network.

**Q4 — user-defined bridge DNS + design implication:**
On `cohort-net`, `dig db` returned `172.20.0.2`, `dig api` returned `172.20.0.3`, and `curl http://api:8080/healthz` returned `ok`. This is different from the default bridge because the user-defined bridge provides container-name DNS. In a 10-service stack, each service can talk to stable service names like `api`, `db`, `redis`, or `worker` instead of hard-coding every other container's IP.

**Q5 — what isolated the stranger + how to bridge it:**
The Docker network boundary isolated the stranger container. `db` was attached to `cohort-net`, while the temporary netshoot container was attached to `cohort-other`, so it could not resolve or reach `db`. The one thing needed to let it reach `db` would be to connect the `db` container to `cohort-other` too with `docker network connect cohort-other db`, or run the stranger container on `cohort-net`.

**Q6 — `-p HOST:CONTAINER` semantics:**
The third probe failed because `18080` is the host-side published port, not the port the API listens on inside the container network. The API process listens on port `8080` inside the container. `-p 18080:8080` creates a host-to-container forwarding rule, usually through Docker's NAT/iptables layer on Linux. Containers on the same Docker network should use the container name and container port, so `api:8080` works but `api:18080` fails.

**Q7 — `docker rm -v` vs named-volume behavior:**
`docker rm -fv throwaway-db` did not delete the named volume `cohort-db-data`; `docker volume ls` still showed it afterward. The `-v` flag on `docker container rm` removes anonymous volumes associated with the container, but named volumes have their own lifecycle and must be removed explicitly with `docker volume rm`. A named volume is different because Docker treats it as a reusable storage object, not just disposable storage attached to one container.

**Q8 — bind vs named volume + why not to poke the mountpoint:**
With a bind mount, I can choose an exact host directory and edit or inspect the files directly from the host, which is useful for local development and sharing source code into a container. I can also mount existing host files or folders into a container. A named volume is better for production app data because Docker manages its lifecycle, it is easier to reuse across container recreation, and the app is less tied to one specific host path. Even though `docker volume inspect` shows a host mountpoint like `/var/lib/docker/volumes/cohort-demo/_data`, manually editing it is risky because Docker and the application expect to manage the data layout, permissions, and consistency.

**Q9 — when tmpfs is the right choice:**
A realistic tmpfs workload is temporary secrets, short-lived scratch files, or sensitive runtime cache that should not be written to disk. tmpfs gives two useful guarantees: it is memory-backed instead of normal disk-backed storage, and the data disappears when the container stops. That makes it useful for temporary data that should not survive container removal.

**Q10 — stopping db before backup + production alternative:**
I stopped the database before backing up the volume to avoid a crash-inconsistent filesystem backup. If Postgres is writing while the raw files are being copied, the backup can capture files from different moments in time and may not restore cleanly. In production, I would use database-aware backup tooling instead of just copying the raw volume, such as `pg_dump`, `pg_basebackup`, WAL archiving, or managed database snapshots.

**Q11 — compose-managed network/volume + external:true:**
Compose created a network named `assignment-04_cohort-net`, using the project directory name as a prefix. This differs from the manually-created `cohort-net` network because Compose namespaces resources per project to avoid collisions. `external: true` on the volume tells Compose to reuse the existing `cohort-db-data` volume instead of creating and owning a new one. This can prevent a real outage because `docker compose down -v` removes Compose-managed volumes, but an external production data volume should not be deleted accidentally.

## 2. Network + volume listing

### `docker network ls`

```text
NETWORK ID     NAME                       DRIVER    SCOPE
4ea63a5b7975   assignment-04_cohort-net   bridge    local
1a6f657ad1e0   bridge                     bridge    local
db5c54f8c4c6   cohort-net                 bridge    local
43812d70ce91   host                       host      local
87c60c851c0d   kind                       bridge    local
ad8a82f25d66   neom-marketplace-net       bridge    local
bd7a4aa7eb97   none                       null      local
```

### `docker network inspect cohort-net --format '{{json .IPAM.Config}}' | jq`

```json
[
  {
    "Subnet": "172.20.0.0/16",
    "Gateway": "172.20.0.1"
  }
]
```

### Compose network inspect

```json
[
  {
    "Subnet": "172.21.0.0/16",
    "Gateway": "172.21.0.1"
  }
]
```

### `docker volume ls`

```text
DRIVER    VOLUME NAME
local     cohort-db-data
local     cohort-demo
```

### `docker volume inspect cohort-db-data`

```json
[
  {
    "CreatedAt": "2026-06-11T01:08:04Z",
    "Driver": "local",
    "Labels": null,
    "Mountpoint": "/var/lib/docker/volumes/cohort-db-data/_data",
    "Name": "cohort-db-data",
    "Options": null,
    "Scope": "local"
  }
]
```

## 3. Files

### `api/Dockerfile`

```dockerfile
# syntax=docker/dockerfile:1.7
FROM python:3.11-slim AS build

RUN python -m venv /opt/venv
ENV PATH=/opt/venv/bin:$PATH

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

FROM python:3.11-slim AS runtime

RUN useradd --uid 1000 --create-home app

COPY --from=build /opt/venv /opt/venv
ENV PATH=/opt/venv/bin:$PATH

WORKDIR /app

COPY app.py .

EXPOSE 8080

USER app

CMD ["gunicorn", "-b", "0.0.0.0:8080", "app:app"]
```

### `api/app.py`

```python
import os, time
import psycopg2
from flask import Flask, request, jsonify

app = Flask(__name__)

DB_HOST = os.environ.get("DB_HOST", "db")
DB_USER = os.environ.get("DB_USER", "cohort")
DB_PASS = os.environ.get("DB_PASS", "cohort")
DB_NAME = os.environ.get("DB_NAME", "cohort")


def connect():
    for _ in range(30):
        try:
            return psycopg2.connect(
                host=DB_HOST, user=DB_USER, password=DB_PASS, dbname=DB_NAME
            )
        except psycopg2.OperationalError:
            time.sleep(1)
    raise RuntimeError("db never came up")


@app.before_request
def ensure_schema():
    if getattr(app, "_ready", False):
        return
    with connect() as c, c.cursor() as cur:
        cur.execute(
            "CREATE TABLE IF NOT EXISTS notes (id SERIAL PRIMARY KEY, body TEXT NOT NULL)"
        )
        c.commit()
    app._ready = True


@app.get("/notes")
def list_notes():
    with connect() as c, c.cursor() as cur:
        cur.execute("SELECT id, body FROM notes ORDER BY id")
        return jsonify([{"id": i, "body": b} for i, b in cur.fetchall()])


@app.post("/notes")
def add_note():
    body = request.json.get("body", "")
    with connect() as c, c.cursor() as cur:
        cur.execute("INSERT INTO notes (body) VALUES (%s) RETURNING id", (body,))
        c.commit()
        return jsonify({"id": cur.fetchone()[0], "body": body}), 201


@app.get("/healthz")
def healthz():
    return ("ok", 200)
```

### `compose.yml`

```yaml
services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: cohort
      POSTGRES_PASSWORD: cohort
      POSTGRES_DB: cohort
    volumes:
      - cohort-db-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U cohort"]
      interval: 5s
      retries: 5
    networks:
      - cohort-net

  api:
    image: cohort-api:0.1.0
    environment:
      DB_HOST: db
    ports:
      - "18080:8080"
    depends_on:
      db:
        condition: service_healthy
    networks:
      - cohort-net

networks:
  cohort-net:

volumes:
  cohort-db-data:
    external: true
```

## 4. Evidence

### Part 1.2 — default bridge DNS gap

```text
--- resolv.conf ---
# Generated by Docker Engine.

nameserver 192.168.65.7

--- dig db ---
--- getent ---
not found
```

### Part 1.3 — hard-coded IP workaround

```text
DB_IP=172.17.0.3
{"body":"hello from the default bridge","id":1}
[{"body":"hello from the default bridge","id":1}]
DB_IP was 172.17.0.3, now 172.17.0.3
```

### Part 2.3 — user-defined bridge DNS works

```text
--- dig db ---
172.20.0.2
--- dig api ---
172.20.0.3
--- curl api ---
ok
```

### Part 2.4 — network inspect connected containers

```text
api 172.20.0.3/16
db 172.20.0.2/16
```

### Part 2.5 — stranger container isolation

```text
--- ping db ---
ping: db: Try again
--- dig db ---
```

### Part 3.1 — published port vs container port

```text
ok (host -> 18080 OK)
ok (container -> 8080 OK)
(port 18080 from container failed — as expected)
```

### Part 4.1 — data loss without named volume

```text
{"body":"this note has no volume protecting it","id":2}
[{"body":"hello from user-defined bridge","id":1},{"body":"this note has no volume protecting it","id":2}]

docker container rm -f db

curl -s http://localhost:18080/notes
500 Internal Server Error

docker container restart api
curl -s http://localhost:18080/notes
[]
```

Note: After recreating `db`, I restarted `api` because the Flask app cached its schema-ready flag. After restarting, `/notes` returned an empty list, proving the old database state was gone.

### Part 4.2/4.3 — named volume protects data

```text
docker volume create cohort-db-data

docker volume inspect cohort-db-data
"Mountpoint": "/var/lib/docker/volumes/cohort-db-data/_data"

{"body":"this note IS protected","id":1}
[{"body":"this note IS protected","id":1}]

docker container rm -f db
docker volume ls --filter name=cohort-db-data

DRIVER    VOLUME NAME
local     cohort-db-data

curl -s http://localhost:18080/notes
[{"body":"this note IS protected","id":1}]
```

### Part 4.3 — `docker rm -fv throwaway-db` did not delete named volume

```text
docker container rm -fv throwaway-db

DRIVER    VOLUME NAME
local     cohort-db-data
```

### Part 4.4 — bind mount vs named volume

```text
-rw-r--r-- 1 root root 22 Jun 10 21:18 note.txt
hello from bind mount

cohort-demo
/var/lib/docker/volumes/cohort-demo/_data
```

### Part 4.5 — tmpfs

```text
-rw-r--r--    1 root     root            10 Jun 11 01:23 x
tmpfs on /scratch type tmpfs (rw,nosuid,nodev,noexec,relatime,size=16384k)
```

### Part 5.1 — backup file exists

```text
-rw-r--r-- 1 root root 6.4M Jun 11 20:44 cohort-db-data-20260611.tar.gz
```

### Part 5.2 — restored volume verified

```text
 id |          body
----+------------------------
  1 | this note IS protected
(1 row)
```

### Part 6 — compose stack works

```text
NAME                  IMAGE                COMMAND                  SERVICE   CREATED         STATUS                   PORTS
assignment-04-api-1   cohort-api:0.1.0     "gunicorn -b 0.0.0.0…"   api       2 minutes ago   Up 2 minutes             0.0.0.0:18080->8080/tcp, [::]:18080->8080/tcp
assignment-04-db-1    postgres:16-alpine   "docker-entrypoint.s…"   db        2 minutes ago   Up 2 minutes (healthy)   5432/tcp

curl -s http://localhost:18080/notes
[{"body":"this note IS protected","id":1}]

api-1  | [2026-06-12 01:03:13 +0000] [1] [INFO] Starting gunicorn 22.0.0
api-1  | [2026-06-12 01:03:13 +0000] [1] [INFO] Listening at: http://0.0.0.0:8080 (1)
api-1  | [2026-06-12 01:03:13 +0000] [1] [INFO] Using worker: sync
api-1  | [2026-06-12 01:03:13 +0000] [7] [INFO] Booting worker with pid: 7
```

## 5. One trade-off I had to make

I chose a named volume for the Postgres data instead of a bind mount. A bind mount would make it easier to inspect files directly from the host, but it would tie the database to a specific host path and increase the risk of permission or manual-edit problems. A named volume is better for app data because Docker manages the lifecycle and the same volume can be reused safely when the database container is recreated.

## 6. One thing I'm still unsure about

I am still unsure about the deeper Linux networking implementation behind Docker's NAT and iptables rules, especially how published ports are translated under the hood.
