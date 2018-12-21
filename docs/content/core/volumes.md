---
title: "Shared volumes"
weight: 3
aliases:
  - /en/master/advanced/volumes/
---

## Quick overview of the volumes options

If option is marked Automatic, then you do not have to do anything for this option to be used.

If option is not Automatic, then you need to enable it manually on the global or project level to use.
All non-automatic options are enabled by placing `DOCKSAL_VOLUMES="<value from volumes column>"` into the respective
`docksal.env` file or using `fin config set` for the same, e.g., `fin config set DOCKSAL_VOLUMES="NFS"`.
Once you set a new volumes option, you must re-create `cli` container. The easiest way is `fin project reset`,
but it will also remove all the data from `db` volume. If you want to retain it, remove `cli` container and start
the project again to recreate it: `fin p remove cli; fin p start`

| OS      | Docker          | Volumes | Automatic | Comments  |
|---------|-----------------|---------|-----------|-----------|
| Linux   | Native          | bind    | **Yes**   | Direct project files access, native FS speed. |
| macOS   | VirtualBox      | nfs     | **Yes**   | `cli` accesses project files from host via NFS. <br> **Pros:** pretty fast, only 10-15% slower than native filesystem. <br> **Cons:** does not support filesystem events (fsnotify). |
| macOS   | Docker for Mac  | bind    | **Yes**   | Files from host are shared with VM via `osxfs`, then `cli` accesses them directly. <br> **Pros:** supports filesystem events and is default for Docker for Mac. <br> **Cons:** pretty slow, 40-50% slower than native filesystem. |
| macOS   | Docker for Mac  | nfs     | *No*      | `cli` accesses project files from host via NFS. <br> **Pros:** much faster than `bind` option. NFS is only 10-15% slower than native filesystem. <br> **Cons:** does not support filesystem events (fsnotify). |
| Windows | ANY             | bind    | **Yes**   | Files from host are shared with VM via SMB, then `cli` accesses them directly. <br> **Pros:** relatively fast, 20% overhead as compared to native FS. <br> **Cons:** does not support filesystem events (fsnotify). |
| ANY     | ANY             | unison  | *No*      | Files from host are shared with VM, but `cli` does not access them directly. Instead, `cli` filesystem is an independent volume, and files are synced from the VM to `cli` via an additional Unison container. Useful for huge projects where FS performance is a bottleneck. <br> **Pros:** maximum `cli` filesystem performance. <br> **Cons:** initial wait for files to sync into `cli`; additional Docksal disk space use; sync delay when you switch git branches; higher CPU usage during files sync; sometimes Unison might 'break.' |
| ANY     | ANY             | none    | *No*      | `none` option is like `unison`, but without the auto-sync. Useful for huge projects where FS performance is a bottleneck, but when `unison` does not work for you. <br> **Pros:** maximum `cli` filesystem performance and no wait for the initial sync. <br> **Cons:** you have to copy files manually or checkout and edit files inside `cli` container only. |

## Full explanation on volumes in Docksal

Let's go over volumes in Docker first. 

In Docker, [volumes](https://docs.docker.com/engine/admin/volumes/volumes/) are used primarily for persisting data 
generated by and used by Docker containers. Docker automatically creates and manages volumes, storing them in a special
location within the host machine's filesystem. There are also different volume plugins, which add support for 
various other data storage backends for volumes.

Mounting an existing folder from the host machine can also be done via volumes using 
[bind mounts](https://docs.docker.com/engine/admin/volumes/bind-mounts/). 

By default, Docksal uses the bind mount approach. 

The VM layer used on macOS/Windows (through VirtualBox or Docker for Mac/Windows) adds some complexity to that, however
that's not something you normally have to worry about. Both Docksal and Docker for Mac/Windows handle that automatically.
 
From the perspective of a container, a local Linux path is mounted regardless of the underlying host OS. 
On Mac, the host filesystem is mounted with NFS, on Windows - using SMB.

Let's take a look at an example.

The host machine is a macOS and the codebase root directory (the "Projects" folder) is `/Users/username/Projects`. 
This directory is mounted with the same path inside the VM: `/Users/username/Projects`. Any path within that directory 
is exactly the same on the host and inside the VM.

When a project stack is started, the project root directory (e.g., `/Users/username/Projects/myproject`) is bind mounted 
into `/var/www` inside the containers. A corresponding line in `docksal.yml` for this would be:

```yaml
version: "2.1"

services:
  cli:
    volumes:
      - ${PROJECT_ROOT}:/var/www:rw
```

`${PROJECT_ROOT}` is automatically set to the project's root directory on the host.

The whole mount chain looks like this (drop the last part for Linux hosts).

```
container ==bind mount==> Linux VM ==NFS/SMB mount==> Mac/Windows host   
```


## Bind volumes

Bind volumes mean native binding of files on host filesystem to Docker. But remember that on macOS and Windows
"host filesystem" for Docker is actually a filesystem of a Virtual Machine it runs in. So while bind volumes
can be used there it actually means that files from your real host has to be mapped into the VM first for
this option to work. On macOS it means mapping via `osxfs:cached`, while on Windows via `SMB`.

Instead of using a host path every time we want to mount a volume, we can give the volume a name and refer to it by name: 

```yaml
version: "2.1"

services:
  cli:
    volumes:
      # Project root volume
      - project_root:/var/www:rw,nocopy
      # Shared ssh-agent socket
      - docksal_ssh_agent:/.ssh-agent:ro
...

volumes:
  project_root:
    driver: local
    driver_opts:
      type: none
      device: ${PROJECT_ROOT}
      o: bind
  docksal_ssh_agent:
    external: true
```

In the example above, `project_root` and `docksal_ssh_agent` are "named volumes". The first one is a project level one,
while the second one is a global volume and is used by all projects.

See [stacks/volumes-bind.yml](https://github.com/docksal/docksal/blob/master/stacks/volumes-bind.yml).

Defining volumes this way makes it much easier to override volume settings in one place (`volumes` section) vs multiple 
places in the yaml file. We can now swap bind mounting with something else. See below.

### osxfs:cached mode with Docker for Mac

As explained above for bind volumes option the files are actually mapped to the VM that Docker runs in
via `osxfs:cached`. Docksal automatically enables the `osxfs:cached` mode on Docker for Mac.

See [stacks/overrides-osxfs.yml](https://github.com/docksal/docksal/blob/master/stacks/overrides-osxfs.yml).


## NFS volumes

With this option `cli` container will map files directly from your real host, rather than mapping them from
the Virtual Machine that Docker runs in.

```yaml
version: "2.1"

volumes:
  cli_home:  # /home/docker volume in cli
  project_root:  # Project root volume (NFS)
    driver: local
    driver_opts:
      type: nfs
      device: :${PROJECT_ROOT}
      o: addr=${DOCKSAL_HOST_IP},vers=3,nolock,noacl,nocto,noatime,nodiratime,actimeo=1
  db_data:  # Database data volume (bind)
  docksal_ssh_agent:  # Shared ssh-agent volume
    external: true
```

See [stacks/volumes-nfs.yml](https://github.com/docksal/docksal/blob/master/stacks/volumes-nfs.yml).

This is what the file sharing chain looks like with a NFS volume. 

```
container:/var/www ==bind mount==> project_root ==> Linux:project_root ==NFS==> macOS:PROJECT_ROOT
```

As you can see, containers mount NFS via the host machine and not directly. This setup method only makes sense on macOS 
with Docker for Mac, for testing and performance comparison purposes.  

### Using NFS volumes

- Add `DOCKSAL_VOLUMES=nfs` either globally in `$HOME/.docksal/docksal.env` or in `.docksal/docksal.env` in a project
- `fin project reset`


## Unison volumes

We can also do more advanced and pretty interesting solutions, like using Unison to synchronize files between the host 
and the `project_root` volume. 

See [stacks/volumes-unison.yml](https://github.com/docksal/docksal/blob/master/stacks/volumes-unison.yml).

Unison volumes make the most sense for Docker for Mac users as an alternative to the (still slow) `osxfs` file sharing.

This is what the file sharing chain looks like with Unison over `osxfs`. 

```
container:/var/www ==bind mount==> project_root <==unison daemon==> Linux:PROJECT_ROOT ==osxfs==> macOS:PROJECT_ROOT
```

`project_root` is a named volume, `PROJECT_ROOT` is a path on the host mounted into the same path in the VM via `osxfs`. 
`unison daemon` does a TWO WAY sync between `PROJECT_ROOT` and `project_root`.

Unlike NFS or SMB, `osxfs` supports `inotify` events, which makes it an ideal option for front-end developers relying on
automatic compilation tools and in-browser live reloading. In the chain above, `inotify` events are not lost and are 
propagated all the way from the macOS host to the container.

The benefits of this setup:

- Full native container file system performance (reads and writes)
- `ionitify` event support
- Low latency (~1s), two way file sync

The downsides:

- Initial sync can take time, especially on large codebases
- Higher disk space usage (double the size of the codebase)
- Additional load from the unison daemon, but nothing compared to the load `osxfs` produces. 

### Using Unison volumes

Unprecedented, native-like FS speed on macOS and Windows (Linux is already native). See [docksal/unison](https://github.com/docksal/unison) for details

- [Install Docksal](/getting-started/setup/)
- Add `DOCKSAL_VOLUMES=unison` in `.docksal/docksal.env` in a project
- `fin project reset`
- Wait until initial sync finishes.


## None volumes

This method is similar to the Unison method, but without the actual sync happening at all.  
Nothing is mounted from the host. An empty `project_root` volume is created and mounted inside containers.

This can be used to provision completely blank environments and have all work (code checkout, etc.) done inside cli.

Provides THE BEST file system performance. Combined with Cloud9, can provide a way of provisioning instant blank 
development environments with the best performance and consistency for Mac and Windows (Linux has the best performance 
naturally). The only added cost is having to stick with a web based IDE and terminal.

See [stacks/volumes-none.yml](https://github.com/docksal/docksal/blob/master/stacks/overrides-none.yml).