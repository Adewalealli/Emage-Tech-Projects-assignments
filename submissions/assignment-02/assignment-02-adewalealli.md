# Assignment 02 — Adewale

**GitHub username:** Adewalealli  
**Date completed:** 2026-05-13  
**Language chosen:** Python

## 1. The image I built

- Final image ID: `sha256:8d72ad63467549df3667f41a8d36d52cc20c473132d60dd6783d553bbd6db129`
- Image size: `186MB disk usage / 45.4MB content size`
- Number of layers: `15`

### Dockerfile

```dockerfile
FROM python:3.11-slim

WORKDIR /app
COPY app.py .

ENV PORT=8000
EXPOSE 8000

CMD ["python", "app.py"]
```

### .dockerignore

```text
.git
.gitignore
node_modules
__pycache__
*.pyc
*.log
README.md
```

## 2. Answers to the 8 questions

**Q1 — what `.dockerignore` affects:**  
It affects the Docker build context, it affects the files Docker sends into the build before the Dockerfile runs. it avoids the files that would not be needed for the build and this makes builds faster, smaller, and safer.

**Q2 — what is the image ID a hash of:**  
The image ID is the finger print of the image. It identifies the actual image Docker built, not just the tag name.

**Q3 — The largest layer and why:**  
The largest layer was the Debian base layer from `python:3.11-slim`, which was `87.4MB` in my output. It was the largest because the base image contains the operating system filesystem and Python runtime, while my own `COPY app.py .` layer was only `12.3kB`.

**Q4 — `--memory 64m` shows up as what value:**  
It showed as `67108864`. Docker stores memory limits in bytes, so 64MB appears as 67,108,864 bytes.

**Q5 — PID of my app inside the container:**  
Using `docker container top greet`, I verified that my app process was `python app.py`. On the Docker host, it showed as PID `3478`. Inside the container, that process is the main process started by `CMD ["python", "app.py"]`, which means it is the container’s PID 1 process. This tells me the container is staying alive because the Python app is running; if `python app.py` exits, the container exits.

**Q6 — `stop` vs `kill`, and which for a database:**  
`docker stop` sends a graceful shutdown signal first, while `docker kill` forcefully terminates the container immediately. For a database, I would use `stop` because the database needs time to flush writes, close files, and shut down cleanly.

**Q7 — what same-IMAGE-ID-across-tags proves:**  
It proves Docker tags are references or labels pointing to an image, not separate copies of the image. `cohort-greet:0.1.0`, `cohort-greet:0.1`, and `cohort-greet:latest` all pointed to the same image ID.

**Q8 — tag vs digest mutability:**  
No, they would not because `alpine:3.19` and `alpine@sha256:...` would not always give the same image if the tag were changed later. A tag is mutable and can be moved, while a digest points to exact immutable image content.

## 3. Evidence

### `docker image ls cohort-greet`

```text
IMAGE                ID             DISK USAGE   CONTENT SIZE   EXTRA
cohort-greet:0.1.0   8d72ad634675        186MB         45.4MB
```

### `docker image history cohort-greet:0.1.0`

```text
IMAGE          CREATED        CREATED BY                                      SIZE      COMMENT
8d72ad634675   21 hours ago   CMD ["python" "app.py"]                         0B        buildkit.dockerfile.v0
<missing>      21 hours ago   EXPOSE [8000/tcp]                               0B        buildkit.dockerfile.v0
<missing>      21 hours ago   ENV PORT=8000                                   0B        buildkit.dockerfile.v0
<missing>      21 hours ago   COPY app.py . # buildkit                        12.3kB    buildkit.dockerfile.v0
<missing>      21 hours ago   WORKDIR /app                                    8.19kB    buildkit.dockerfile.v0
<missing>      5 days ago     CMD ["python3"]                                 0B        buildkit.dockerfile.v0
<missing>      5 days ago     RUN /bin/sh -c set -eux;  for src in idle3 p…   16.4kB    buildkit.dockerfile.v0
<missing>      5 days ago     RUN /bin/sh -c set -eux;   savedAptMark="$(a…   48.4MB    buildkit.dockerfile.v0
<missing>      5 days ago     ENV PYTHON_SHA256=272179ddd9a2e41a0fc8e42e33…   0B        buildkit.dockerfile.v0
<missing>      5 days ago     ENV PYTHON_VERSION=3.11.15                      0B        buildkit.dockerfile.v0
<missing>      5 days ago     ENV GPG_KEY=A035C8C19219BA821ECEA86B64E628F8…   0B        buildkit.dockerfile.v0
<missing>      5 days ago     RUN /bin/sh -c set -eux;  apt-get update;  a…   4.94MB    buildkit.dockerfile.v0
<missing>      5 days ago     ENV LANG=C.UTF-8                                0B        buildkit.dockerfile.v0
<missing>      5 days ago     ENV PATH=/usr/local/bin:/usr/local/sbin:/usr…   0B        buildkit.dockerfile.v0
<missing>      8 days ago     # debian.sh --arch 'amd64' out/ 'trixie' '@1…   87.4MB    debuerreotype 0.17
```

### Full image ID

```text
sha256:8d72ad63467549df3667f41a8d36d52cc20c473132d60dd6783d553bbd6db129
```

### Detached run command

```bash
docker container run -d \
  --name greet \
  -p 8080:8000 \
  -e STUDENT_NAME="Adewale" \
  -e GREETING="hi" \
  --restart unless-stopped \
  --memory 64m \
  --cpus 0.25 \
  cohort-greet:0.1.0
```

Output:

```text
b2fed3e518ba9202e32e72c1380232f1eb3d6ce92afefbcf9d7d48eb279d1522
```

### `curl http://localhost:8080`

```text
hi, Adewale — 2026-05-13T21:53:35.401668Z
hi, Adewale — 2026-05-13T21:53:35.408396Z
```

### `docker container logs greet`

```text
listening on :8000
[req] 172.17.0.1 "GET / HTTP/1.1" 200 -
[req] 172.17.0.1 "GET / HTTP/1.1" 200 -
```

### `docker container stats --no-stream greet`

```text
CONTAINER ID   NAME      CPU %     MEM USAGE / LIMIT   MEM %     NET I/O           BLOCK I/O         PIDS
b2fed3e518ba   greet     0.01%     14.8MiB / 64MiB     23.13%    2.74kB / 1.34kB   16.1MB / 1.95MB   1
```

### `docker container inspect -f '{{.HostConfig.RestartPolicy.Name}} {{.HostConfig.Memory}}' greet`

```text
unless-stopped 67108864
```

### Environment check inside container

```text
GPG_KEY=A035C8C19219BA821ECEA86B64E628F8D684696D
GREETING=hi
HOME=/root
HOSTNAME=b2fed3e518ba
LANG=C.UTF-8
PATH=/usr/local/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
PORT=8000
PWD=/app
PYTHON_SHA256=272179ddd9a2e41a0fc8e42e33dfbdca0b3711aa5abf372d3f2d51543d09b625
PYTHON_VERSION=3.11.15
STUDENT_NAME=Adewale
TERM=xterm
```

### File check inside container

```text
# ls /app
app.py
```

### `docker container top greet`

```text
UID                 PID                 PPID                C                   STIME               TTY                 TIME                CMD
root                3478                3455                0                   21:53               ?                   00:00:00            python app.py
```

### `docker image ls cohort-greet` after adding multiple tags

```text
IMAGE                 ID             DISK USAGE   CONTENT SIZE   EXTRA
cohort-greet:0.1      8d72ad634675        186MB         45.4MB    U
cohort-greet:0.1.0    8d72ad634675        186MB         45.4MB    U
cohort-greet:latest   8d72ad634675        186MB         45.4MB    U
```

### Pulling Alpine by tag and digest

```text
3.19: Pulling from library/alpine
Digest: sha256:6baf43584bcb78f2e5847d1de515f23499913ac9f12bdf834811a3145eb11ca1
Status: Image is up to date for alpine:3.19
docker.io/library/alpine:3.19
```

### `docker image inspect alpine:3.19 -f '{{index .RepoDigests 0}}'`

```text
alpine@sha256:6baf43584bcb78f2e5847d1de515f23499913ac9f12bdf834811a3145eb11ca1
```

### Pulling Alpine by digest

```text
docker.io/library/alpine@sha256:6baf43584bcb78f2e5847d1de515f23499913ac9f12bdf834811a3145eb11ca1: Pulling from library/alpine
Digest: sha256:6baf43584bcb78f2e5847d1de515f23499913ac9f12bdf834811a3145eb11ca1
Status: Image is up to date for alpine@sha256:6baf43584bcb78f2e5847d1de515f23499913ac9f12bdf834811a3145eb11ca1
docker.io/library/alpine@sha256:6baf43584bcb78f2e5847d1de515f23499913ac9f12bdf834811a3145eb11ca1
```

### `docker image ls --digests alpine`

```text
REPOSITORY   TAG       DIGEST                                                                    IMAGE ID       CREATED        SIZE
alpine       latest    sha256:5b10f432ef3da1b8d4c7eb6c487f2f5a8f096bc91145e68878dd4a5019afde11   5b10f432ef3d   4 weeks ago    13.1MB
alpine       3.19      sha256:6baf43584bcb78f2e5847d1de515f23499913ac9f12bdf834811a3145eb11ca1   6baf43584bcb   7 months ago   11.6MB
```

### Cleanup

```text
Untagged: cohort-greet:latest
Untagged: cohort-greet:0.1
Untagged: cohort-greet:0.1.0
Deleted: sha256:8d72ad63467549df3667f41a8d36d52cc20c473132d60dd6783d553bbd6db129
```

### `docker system df`

```text
TYPE            TOTAL     ACTIVE    SIZE      RECLAIMABLE
Images          14        2         2.85GB    796.4MB (27%)
Containers      4         3         3.94MB    20.48kB (0%)
Local Volumes   4         4         3.141GB   0B (0%)
Build Cache     43        0         452.9MB   251.2MB
```

## 4. One thing that surprised me

One thing that i understand now that was confusing before this class, was the purpose of .dockerignore

## 5. One thing I'm still unsure about

how to pick the right base image in production, what kind of factors to consider before doing so. 
