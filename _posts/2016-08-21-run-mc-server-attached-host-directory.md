---
layout: post
title:  "Run a Minecraft server container with attached host directory"

---

Attaching host directories to a Docker container instance and getting the read/write permissions correct can actually be a little tricky. By default attached directories are accessed as the same user running the Docker daemon, which is usually `root`.

My [Minecraft server image](https://hub.docker.com/r/itzg/minecraft-server/) has a security feature where the java process runs as a non-root user (UID=1000 by default); however, that adds another level of complexity to attached host directory access if your current user isn't set at user ID 1000. 

To counteract that possible mismatch environment variable options are provided to define the user ID and group ID of the process that will run java and access the `/data` attach point:

* `-e UID=uid`
* `-e GID=gid`

## Hands-on Example

First, setup a host directory owned and modifiable by your current user:

```
sudo mkdir /minecraft
sudo chown -R $(id -un) /minecraft
```

At this point the directory is empty, but we can startup a new container and let it populate that directory with defaults:

```
docker run -d \
  --name mc-attached-data \
  -v /minecraft:/data \
  -e UID=$(id -u) -e GID=$(id -g) \
  -e EULA=TRUE \
  -p 25565:25565 \
  itzg/minecraft-server
```

From the host-side you can browse the active Minecraft data files:

```
ls /minecraft
```

You can even stop and remove the container and the populated host directory remains intact. At this point you can adjust the `server.properties`, replace the world data with one you downloaded, etc.

```
docker stop mc-attached-data
docker rm mc-attached-data

vi /minecraft/server.properties
```

At any time after that you can start up a fresh new container, attach the same directory to `/data`, and this server instance will use that previously configured data directory. In the following example I gave the container a distinct name, "mc-reuse".

```
docker run -d \
  --name mc-reuse \
  -v /minecraft:/data \
  -e UID=$(id -u) -e GID=$(id -g) \
  -e EULA=TRUE \
  -p 25565:25565 \
  itzg/minecraft-server
```

