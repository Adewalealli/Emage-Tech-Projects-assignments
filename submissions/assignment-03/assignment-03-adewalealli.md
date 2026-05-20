# Assignment 03 — Adewale Alli

**GitHub username:** Adewalealli  
**Date completed:** 2026-05-19  
**Git SHA of submitted app:** 053a48c  

## 1. Size comparison table

| Variant | Size | Layers | Stop time | Exit code |
|---|---:|---:|---:|---:|
| `cohort-greet:naive` | 1.62GB | 18 | 0m5.369s | 137 |
| `cohort-greet:multi` | 238MB | 19 | 0m0.445s | 0 |

Layers were counted with `docker image history <tag> | wc -l` minus 1 for the header.

## 2. Final image digest

`sha256:403cfb5f498a334ef6cd1373edfe02adb42ae25a3a72d824480d5bf5b1f7d508`

## 3. Answers to the 7 questions

**Q1 — naive size + stop behaviour + why:**

The naive image size was 1.62GB. When I stopped the container with `docker container stop --timeout 5`, it took 0m5.369s and exited with `ExitCode=137`.

The reason is that the naive Dockerfile used shell-form CMD:

```dockerfile
CMD gunicorn -b 0.0.0.0:8080 app:app
```

Docker runs shell-form commands through `/bin/sh -c`, which means the shell becomes PID 1 instead of Gunicorn running directly. When Docker sent SIGTERM, the application did not shut down cleanly through the shell wrapper, so Docker waited for the timeout and then force-killed it with SIGKILL. That produced exit code 137.

**Q2 — build output, CACHED vs rebuilt:**

After adding a trivial edit to `app.py`, the rebuild showed that `COPY requirements.txt .` was cached and the `RUN pip install --no-cache-dir -r requirements.txt` layer was also cached:

```text
#8 [build 4/5] COPY requirements.txt .
#8 CACHED
#13 [build 5/5] RUN pip install --no-cache-dir -r requirements.txt
#13 CACHED
#15 [runtime 5/5] COPY app.py .
```

The dependency layers were cached because `requirements.txt` did not change. Only the `COPY app.py .` layer needed to rebuild because the app file changed. This proves the Dockerfile is ordered correctly: dependencies are copied and installed before application code, so normal app edits do not bust the dependency install layer.

**Q3 — new stop time/exit + which change:**

The multi-stage image stopped in 0m0.445s and exited with `ExitCode=0`. Compared to the naive image, this is much better than 0m5.369s and `ExitCode=137`.

The Dockerfile change responsible was switching from shell-form CMD to exec-form CMD:

```dockerfile
CMD ["gunicorn", "-b", "0.0.0.0:8080", "app:app"]
```

Exec form runs Gunicorn directly instead of wrapping it in `/bin/sh -c`. This lets Docker’s SIGTERM reach the main process correctly, so the application can shut down cleanly.

**Q4 — size reduction breakdown:**

The naive image was 1.62GB, which is about 1658.88MB. The multi-stage image was 238MB. That is a reduction of about 85.65%.

The biggest savings came from switching from `python:3.11` to `python:3.11.10-slim`, which removed a lot of unnecessary operating system and build content from the base image. The multi-stage build also separated the build environment from the runtime environment. Dependencies were installed into `/opt/venv` in the build stage, and only `/opt/venv` plus `app.py` were copied into the runtime stage. The `.dockerignore` also helped keep Git data, markdown files, environment files, cache files, and extra Dockerfiles out of the build context.

Specific layers that matter in the final image include the copied virtual environment layer:

```text
COPY /opt/venv /opt/venv # buildkit   34.2MB
```

and the app copy layer:

```text
COPY app.py . # buildkit              12.3kB
```

The runtime image avoids carrying the whole original project and avoids the larger full Python base image used by the naive Dockerfile.

**Q5 — cache-mount timings + CI relevance:**

The cold build took:

```text
real    0m18.111s
```

The warm cache-mount build took:

```text
real    0m18.400s
```

In this small Flask app, the warm cache mount did not produce a speed improvement. The second build was slightly slower. That is still useful evidence because the measurement shows the real behavior of this workload. In a CI pipeline, this matters more when the dependency set is larger. CI runners often have a cold normal Docker layer cache, but a persisted BuildKit cache mount can reuse downloaded packages across builds and reduce dependency install time.

**Q6 — secret marker + what `ARG` would leak:**

The marker output was:

```text
412f
```

The leak check output was:

```text
no leak
```

The marker file proves the BuildKit secret was readable during the build, but only the first 4 characters were written into the image as proof. The `no leak` result shows the full token did not appear in Docker image history.

If I had used `ARG PYPI_TOKEN`, the token could have leaked through the image build record, image history, or build metadata. BuildKit secret mounts are safer because the secret is mounted temporarily only for that `RUN` instruction and is not saved into a normal image layer.

**Q7 — tag vs digest for k8s manifest:**

For a production Kubernetes manifest, I would usually use a version plus Git SHA tag, such as:

```text
cohort-greet:0.1.0-053a48c
```

That is readable and traceable because it tells me both the application version and the exact Git commit. However, if the security team requires exact reproducibility, I would pin by digest instead:

```text
cohort-greet@sha256:403cfb5f498a334ef6cd1373edfe02adb42ae25a3a72d824480d5bf5b1f7d508
```

A tag is a pointer that can theoretically be moved. A digest identifies the exact image content. If the image content changes, the digest changes.

## 4. Files

### Final `Dockerfile`

```dockerfile
# syntax=docker/dockerfile:1.7

# ── build stage ──
FROM python:3.11.10-slim AS build

RUN python -m venv /opt/venv
ENV PATH=/opt/venv/bin:$PATH

WORKDIR /app
COPY requirements.txt .
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install --no-cache-dir -r requirements.txt

# ── runtime stage ──
FROM python:3.11.10-slim AS runtime

COPY --from=build /opt/venv /opt/venv
ENV PATH=/opt/venv/bin:$PATH

WORKDIR /app

RUN useradd --uid 1000 --create-home app \
    && chown -R app:app /app

COPY app.py .

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8080/healthz')"

USER app

CMD ["gunicorn", "-b", "0.0.0.0:8080", "app:app"]
```

### `Dockerfile.naive`

```dockerfile
FROM python:3.11
WORKDIR /app
COPY . .
RUN pip install -r requirements.txt
EXPOSE 8080
CMD gunicorn -b 0.0.0.0:8080 app:app
```

### `Dockerfile.secret`

```dockerfile
# syntax=docker/dockerfile:1.7

FROM python:3.11.10-slim AS build

RUN python -m venv /opt/venv
ENV PATH=/opt/venv/bin:$PATH

WORKDIR /app
COPY requirements.txt .

RUN --mount=type=secret,id=pypi_token \
    token="$(cat /run/secrets/pypi_token)" \
    && printf "%s\n" "$token" | cut -c1-4 > /where-token-was-used \
    && pip install --no-cache-dir -r requirements.txt

FROM python:3.11.10-slim AS runtime

COPY --from=build /opt/venv /opt/venv
COPY --from=build /where-token-was-used /where-token-was-used
ENV PATH=/opt/venv/bin:$PATH

WORKDIR /app

RUN useradd --uid 1000 --create-home app \
    && chown -R app:app /app

COPY app.py .

EXPOSE 8080

USER app

CMD ["gunicorn", "-b", "0.0.0.0:8080", "app:app"]
```

### `.dockerignore`

```text
.git/
.gitignore
__pycache__/
*.pyc
Dockerfile*
*.md
.env*
```

## 5. Evidence

### `docker image ls cohort-greet`

```text
REPOSITORY     TAG             IMAGE ID       SIZE
cohort-greet   secret          88aaa59362a4   238MB
cohort-greet   0.1.0           403cfb5f498a   238MB
cohort-greet   0.1.0-053a48c   403cfb5f498a   238MB
cohort-greet   git-053a48c     403cfb5f498a   238MB
cohort-greet   multi           403cfb5f498a   238MB
cohort-greet   naive           9792711c828b   1.62GB
```

### `docker image history cohort-greet:multi`

```text
IMAGE          CREATED         CREATED BY                                      SIZE      COMMENT
403cfb5f498a   4 minutes ago   CMD ["gunicorn" "-b" "0.0.0.0:8080" "app:app…   0B        buildkit.dockerfile.v0
<missing>      4 minutes ago   USER app                                        0B        buildkit.dockerfile.v0
<missing>      4 minutes ago   HEALTHCHECK &{["CMD-SHELL" "python -c \"impo…   0B        buildkit.dockerfile.v0
<missing>      4 minutes ago   EXPOSE map[8080/tcp:{}]                         0B        buildkit.dockerfile.v0
<missing>      4 minutes ago   COPY app.py . # buildkit                        12.3kB    buildkit.dockerfile.v0
<missing>      4 minutes ago   RUN /bin/sh -c useradd --uid 1000 --create-h…   73.7kB    buildkit.dockerfile.v0
<missing>      4 minutes ago   WORKDIR /app                                    8.19kB    buildkit.dockerfile.v0
<missing>      4 minutes ago   ENV PATH=/opt/venv/bin:/usr/local/bin:/usr/l…   0B        buildkit.dockerfile.v0
<missing>      4 minutes ago   COPY /opt/venv /opt/venv # buildkit             34.2MB    buildkit.dockerfile.v0
```

### Secret marker

Command:

```bash
docker container run --rm cohort-greet:secret cat /where-token-was-used
```

Output:

```text
412f
```

### Secret leak check

Command:

```bash
docker image history --no-trunc cohort-greet:secret | grep -i "$PYPI_TOKEN" \
  && echo "LEAKED" || echo "no leak"
```

Output:

```text
no leak
```

### Hadolint

Command:

```bash
docker container run --rm -i hadolint/hadolint < Dockerfile
```

Output:

```text

```

No output means the final Dockerfile passed hadolint with no warnings.

### Cache timing

Cold build:

```text
real    0m18.111s
user    0m0.172s
sys     0m0.325s
```

Warm cache-mount build:

```text
real    0m18.400s
user    0m0.160s
sys     0m0.385s
```

## 6. One trade-off I had to make

I chose `python:3.11.10-slim` instead of Alpine or distroless. The slim image is smaller than the full `python:3.11` image but still familiar and compatible with Python packages without needing Alpine-specific debugging. Alpine could be smaller, but it can introduce musl libc compatibility issues. Distroless could be even tighter for production, but it would make debugging harder and would require more careful runtime setup.

## 7. One thing I'm still unsure about

I am still unsure how much BuildKit cache mounts help in larger CI pipelines compared to normal Docker layer caching.
