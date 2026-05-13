Windows PowerShell
Copyright (C) Microsoft Corporation. All rights reserved.

Install the latest PowerShell for new features and improvements! https://aka.ms/PSWindows

PS C:\Windows\system32> cd ~/assignment-02
cd : Cannot find path 'C:\Users\StarLord\assignment-02' because it does not exist.
At line:1 char:1
+ cd ~/assignment-02
+ ~~~~~~~~~~~~~~~~~~
    + CategoryInfo          : ObjectNotFound: (C:\Users\StarLord\assignment-02:String) [Set-Location], ItemNotFoundExc
   eption
    + FullyQualifiedErrorId : PathNotFound,Microsoft.PowerShell.Commands.SetLocationCommand

PS C:\Windows\system32> wsl
starlord@NeoM-Legion:/mnt/c/Windows/system32$ cd ~/assignment-02
ls -la
total 16
drwxr-xr-x  2 starlord starlord 4096 May 12 21:14 .
drwxr-x--- 19 starlord starlord 4096 May 12 20:43 ..
-rw-r--r--  1 starlord starlord  103 May 12 21:06 Dockerfile
-rw-r--r--  1 starlord starlord  742 May 12 21:14 app.py
starlord@NeoM-Legion:~/assignment-02$ cat > .dockerignore <<'EOF'
.git
.gitignore
node_modules
__pycache__
*.pyc
*.log
README.md
EOF
starlord@NeoM-Legion:~/assignment-02$ docker image build -t cohort-greet:0.1.0 .

The command 'docker' could not be found in this WSL 2 distro.
We recommend to activate the WSL integration in Docker Desktop settings.

For details about using Docker Desktop with WSL 2, visit:

https://docs.docker.com/go/wsl2/

starlord@NeoM-Legion:~/assignment-02$ docker desktop

The command 'docker' could not be found in this WSL 2 distro.
We recommend to activate the WSL integration in Docker Desktop settings.

For details about using Docker Desktop with WSL 2, visit:

https://docs.docker.com/go/wsl2/

starlord@NeoM-Legion:~/assignment-02$ exit
logout
PS C:\Windows\system32> docker desktop
Usage:  docker desktop COMMAND

The Docker Desktop CLI lets you perform key operations such as starting, stopping,
restarting, and updating Docker Desktop directly from the command line.

Management Commands:
  disable      Disable a feature
  enable       Enable a feature
  engine       Commands to list and switch engine modes (Windows only)
  kubernetes   Manage Kubernetes

Commands:
  diagnose     Diagnose Docker Desktop
  logs         Print log entries
  restart      Restart Docker Desktop
  start        Start Docker Desktop
  status       Show the status of the Docker Desktop engines
  stop         Stop Docker Desktop
  update       Manage Docker Desktop updates
  version      Show the Docker Desktop CLI plugin version information

Run 'docker desktop COMMAND --help' for more information on a command.
PS C:\Windows\system32> docker --version
Docker version 29.4.1, build 055a478
PS C:\Windows\system32> docker ps
failed to connect to the docker API at npipe:////./pipe/dockerDesktopLinuxEngine; check if the path is correct and if the daemon is running: open //./pipe/dockerDesktopLinuxEngine: The system cannot find the file specified.
PS C:\Windows\system32> start "" "C:\Program Files\Docker\Docker\Docker Desktop.exe"
Start-Process : Cannot validate argument on parameter 'FilePath'. The argument is null or empty. Provide an argument
that is not null or empty, and then try the command again.
At line:1 char:7
+ start "" "C:\Program Files\Docker\Docker\Docker Desktop.exe"
+       ~~
    + CategoryInfo          : InvalidData: (:) [Start-Process], ParameterBindingValidationException
    + FullyQualifiedErrorId : ParameterArgumentValidationError,Microsoft.PowerShell.Commands.StartProcessCommand

PS C:\Windows\system32> Start-Process "C:\Program Files\Docker\Docker\Docker Desktop.exe"
PS C:\Windows\system32> docker ps
CONTAINER ID   IMAGE                  COMMAND                  CREATED       STATUS              PORTS                       NAMES
499b8f071219   kindest/node:v1.32.0   "/usr/local/bin/entr…"   2 weeks ago   Up About a minute                               neom-k8s-worker2
3fa70cfbd026   kindest/node:v1.32.0   "/usr/local/bin/entr…"   2 weeks ago   Up About a minute   127.0.0.1:42885->6443/tcp   neom-k8s-control-plane
af3414a8c6fb   kindest/node:v1.32.0   "/usr/local/bin/entr…"   2 weeks ago   Up About a minute                               neom-k8s-worker
PS C:\Windows\system32> wsl
starlord@NeoM-Legion:/mnt/c/Windows/system32$ cd ~/assignment-02
starlord@NeoM-Legion:~/assignment-02$ docker image build -t cohort-greet:0.1.0 .
[+] Building 0.9s (9/9) FINISHED                                                                         docker:default
 => [internal] load build definition from Dockerfile                                                               0.0s
 => => transferring dockerfile: 140B                                                                               0.0s
 => [internal] load metadata for docker.io/library/python:3.11-slim                                                0.7s
 => [auth] library/python:pull token for registry-1.docker.io                                                      0.0s
 => [internal] load .dockerignore                                                                                  0.0s
 => => transferring context: 103B                                                                                  0.0s
 => [1/3] FROM docker.io/library/python:3.11-slim@sha256:9a7765b36773a37061455b332f18e265e7f58f6fea9c419a550d2a8b  0.0s
 => => resolve docker.io/library/python:3.11-slim@sha256:9a7765b36773a37061455b332f18e265e7f58f6fea9c419a550d2a8b  0.0s
 => [internal] load build context                                                                                  0.0s
 => => transferring context: 28B                                                                                   0.0s
 => CACHED [2/3] WORKDIR /app                                                                                      0.0s
 => CACHED [3/3] COPY app.py .                                                                                     0.0s
 => exporting to image                                                                                             0.1s
 => => exporting layers                                                                                            0.0s
 => => exporting manifest sha256:752c97a577e5d87981680e6d85ca1e30de96a56fd10054d35bcfd9efec8d3604                  0.0s
 => => exporting config sha256:cb236ed6aa7639e1c4ecdd464ef6ef9fc273da3a45b28f442eb1ba39e8d2b627                    0.0s
 => => exporting attestation manifest sha256:bd7769e58759c645b2ac37a754d2e81e4b4e2a7802779389dbaebd535580ac7c      0.0s
 => => exporting manifest list sha256:8d72ad63467549df3667f41a8d36d52cc20c473132d60dd6783d553bbd6db129             0.0s
 => => naming to docker.io/library/cohort-greet:0.1.0                                                              0.0s
 => => unpacking to docker.io/library/cohort-greet:0.1.0                                                           0.0s
starlord@NeoM-Legion:~/assignment-02$ docker image ls cohort-greet
                                                                                                    i Info →   U  In Use
IMAGE                ID             DISK USAGE   CONTENT SIZE   EXTRA
cohort-greet:0.1.0   8d72ad634675        186MB         45.4MB
starlord@NeoM-Legion:~/assignment-02$ docker image history cohort-greet:0.1.0
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
starlord@NeoM-Legion:~/assignment-02$ docker image inspect -f '{{.Id}}' cohort-greet:0.1.0
sha256:8d72ad63467549df3667f41a8d36d52cc20c473132d60dd6783d553bbd6db129
starlord@NeoM-Legion:~/assignment-02$ docker container rm -f greet 2>/dev/null
starlord@NeoM-Legion:~/assignment-02$ docker container run -d \
  --name greet \
  -p 8080:8000 \
  -e STUDENT_NAME="Adewale" \
  -e GREETING="hi" \
  --restart unless-stopped \
  --memory 64m \
  --cpus 0.25 \
  cohort-greet:0.1.0
b2fed3e518ba9202e32e72c1380232f1eb3d6ce92afefbcf9d7d48eb279d1522
starlord@NeoM-Legion:~/assignment-02$ curl http://localhost:8080
curl http://localhost:8080
hi, Adewale — 2026-05-13T21:53:35.401668Z
hi, Adewale — 2026-05-13T21:53:35.408396Z
starlord@NeoM-Legion:~/assignment-02$ docker container logs greet
listening on :8000
[req] 172.17.0.1 "GET / HTTP/1.1" 200 -
[req] 172.17.0.1 "GET / HTTP/1.1" 200 -
starlord@NeoM-Legion:~/assignment-02$ docker container stats --no-stream greet
CONTAINER ID   NAME      CPU %     MEM USAGE / LIMIT   MEM %     NET I/O           BLOCK I/O         PIDS
b2fed3e518ba   greet     0.01%     14.8MiB / 64MiB     23.13%    2.74kB / 1.34kB   16.1MB / 1.95MB   1
starlord@NeoM-Legion:~/assignment-02$ docker container inspect -f '{{.HostConfig.RestartPolicy.Name}} {{.HostConfig.Memory}}' greet
unless-stopped 67108864
starlord@NeoM-Legion:~/assignment-02$ docker container exec -it greet sh
# env | sort
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
# ls /app
app.py
# exit
starlord@NeoM-Legion:~/assignment-02$ docker container top greet
UID                 PID                 PPID                C                   STIME               TTY                 TIME                CMD
root                3478                3455                0                   21:53               ?                   00:00:00            python app.py
starlord@NeoM-Legion:~/assignment-02$ docker image tag cohort-greet:0.1.0 cohort-greet:latest
starlord@NeoM-Legion:~/assignment-02$ docker image tag cohort-greet:0.1.0 cohort-greet:0.1
starlord@NeoM-Legion:~/assignment-02$ docker image ls cohort-greet
                                                                                                    i Info →   U  In Use
IMAGE                 ID             DISK USAGE   CONTENT SIZE   EXTRA
cohort-greet:0.1      8d72ad634675        186MB         45.4MB    U
cohort-greet:0.1.0    8d72ad634675        186MB         45.4MB    U
cohort-greet:latest   8d72ad634675        186MB         45.4MB    U
starlord@NeoM-Legion:~/assignment-02$ docker image pull alpine:3.19
3.19: Pulling from library/alpine
Digest: sha256:6baf43584bcb78f2e5847d1de515f23499913ac9f12bdf834811a3145eb11ca1
Status: Image is up to date for alpine:3.19
docker.io/library/alpine:3.19
starlord@NeoM-Legion:~/assignment-02$ docker image inspect alpine:3.19 -f '{{index .RepoDigests 0}}'
alpine@sha256:6baf43584bcb78f2e5847d1de515f23499913ac9f12bdf834811a3145eb11ca1
starlord@NeoM-Legion:~/assignment-02$ DIGEST=$(docker image inspect alpine:3.19 -f '{{index .RepoDigests 0}}')
starlord@NeoM-Legion:~/assignment-02$ docker image pull "$DIGEST"
docker.io/library/alpine@sha256:6baf43584bcb78f2e5847d1de515f23499913ac9f12bdf834811a3145eb11ca1: Pulling from library/alpine
Digest: sha256:6baf43584bcb78f2e5847d1de515f23499913ac9f12bdf834811a3145eb11ca1
Status: Image is up to date for alpine@sha256:6baf43584bcb78f2e5847d1de515f23499913ac9f12bdf834811a3145eb11ca1
docker.io/library/alpine@sha256:6baf43584bcb78f2e5847d1de515f23499913ac9f12bdf834811a3145eb11ca1
starlord@NeoM-Legion:~/assignment-02$ docker image ls --digests alpine
REPOSITORY   TAG       DIGEST                                                                    IMAGE ID       CREATED        SIZE
alpine       latest    sha256:5b10f432ef3da1b8d4c7eb6c487f2f5a8f096bc91145e68878dd4a5019afde11   5b10f432ef3d   4 weeks ago    13.1MB
alpine       3.19      sha256:6baf43584bcb78f2e5847d1de515f23499913ac9f12bdf834811a3145eb11ca1   6baf43584bcb   7 months ago   11.6MB
starlord@NeoM-Legion:~/assignment-02$ 7. Cleanup
docker container rm -f greet 2>/dev/null
7.: command not found
greet
starlord@NeoM-Legion:~/assignment-02$ 7. Cleanup
docker container rm -f greet 2>/dev/nul^C
starlord@NeoM-Legion:~/assignment-02$ docker container rm -f greet 2>/dev/null
starlord@NeoM-Legion:~/assignment-02$ docker image rm cohort-greet:latest cohort-greet:0.1 cohort-greet:0.1.0 2>/dev/null
Untagged: cohort-greet:latest
Untagged: cohort-greet:0.1
Untagged: cohort-greet:0.1.0
Deleted: sha256:8d72ad63467549df3667f41a8d36d52cc20c473132d60dd6783d553bbd6db129
starlord@NeoM-Legion:~/assignment-02$ docker system df
TYPE            TOTAL     ACTIVE    SIZE      RECLAIMABLE
Images          14        2         2.85GB    796.4MB (27%)
Containers      4         3         3.94MB    20.48kB (0%)
Local Volumes   4         4         3.141GB   0B (0%)
Build Cache     43        0         452.9MB   251.2MB
starlord@NeoM-Legion:~/assignment-02$ cd
starlord@NeoM-Legion:~$ cd ~/kubernetes-january-2026-cohort
-bash: cd: /home/starlord/kubernetes-january-2026-cohort: No such file or directory
starlord@NeoM-Legion:~$



