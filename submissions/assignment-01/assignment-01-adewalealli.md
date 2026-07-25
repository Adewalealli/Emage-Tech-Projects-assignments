# Assignment 01 — CLI Essentials & Docker Commands

## Part 1 Reflection

One CLI command that was new to me was `wc -l`. I used it to count the number of lines inside `commands.txt` and confirm that I had exactly five Docker subcommands listed. I also practiced using `grep` with a pipe to filter command output. This matters because Docker and Kubernetes troubleshooting often requires reading logs, filtering output, and finding the exact information I need from the terminal.

## Part 2 Answers

### 1. Images vs Containers

After pulling and running Alpine, I had at least one image, `alpine:3.19`, and one container created from that image.

The image is the read-only template. The container is the running or stopped instance created from the image. The Alpine container stopped because the command `echo hi` finished, but the stopped container still exists until I remove it.

### 2. Difference Between `docker run -it alpine sh` and `docker exec -it <name> sh`

`docker run -it alpine sh` creates a brand-new container from the Alpine image and opens a shell inside it.

`docker exec -it <name> sh` opens a shell inside a container that is already running.

I would use `docker run` when I want to create a new container. I would use `docker exec` when I want to inspect or troubleshoot a container that is already running.

### 3. Read-only Mount Flag

The flag used to mount a file or path into a container is `--mount`.

To make it read-only, I can add `readonly` to the mount options.


I confirmed the nginx container was running with: docker container ls     Name:Neom-legion-practice

It used the nginx:1.25 image and mapped host port 8081 to container port 80.  Opened a browser @ http://localhost:8081

After editing the nginx homepage inside the container, the browser showed: Hello from StarLorD

I used this command to print only the container IP address: docker container inspect -f '{{.NetworkSettings.IPAddress}}' Neom-legion-practice
[i forgot to save the ip before deleting it]

One Thing That Surprised Me-
[ That changing a file inside the container can immediately change what the browser displays.]
