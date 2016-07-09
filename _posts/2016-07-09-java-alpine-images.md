---
layout: post
title:  "Lessons learned switching to Java Alpine Docker base image"

---


Somewhere around the [Java 8u92](https://hub.docker.com/r/library/java/tags/8u92-jre-alpine/) versions, the [official Docker images for Java](https://hub.docker.com/r/_/java/) started providing Alpine variants of the base images. I believe the benefits of that option are smaller footprint and smaller security attack surface.

Both are fantastic benefits, but there's some gotchas if you're used to a Debian (or similar) base image.

### Doesn't include curl or wget

For the reasons above, the Alpine variant doesn't come with `curl` or `wget`. Avoid the temptation to use `apk` to install those packages and instead take this as a reminder to better leverage Docker. 

The `ADD` Dockerfile instruction already supports source files retrieved via URL. So rather than using `RUN` to issue a curl/wget, just add a file from a URL natively, such as

```
ADD https://somehost/somefile.zip /tmp/somefile.tgz
```

### useradd is adduser

If your `Dockerfile` or scripts are using the command `useradd` (from Debian land), then use `adduser` instead. **BUT**, check out the next couple of sections.

### Default system user has no shell

Creating a system user with

    adduser -S sysuser

will result in a user than can't be used with an `su -c` invocation since no shell is configured by default. Instead explicitly set a shell, such as

    adduser -S -s /bin/sh sysuser

### Use su to drop privileges

Since `sudo` isn't installed by default and we want to minimize adding more packages (to lower security exposure), then use `su` instead. To run a command as a system user (with an explicit shell configured as above), use something like

    su -c "command" - sysuser

