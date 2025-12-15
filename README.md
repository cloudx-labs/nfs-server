# gcp-nfs-server

Maintained rebuild of the old GoogleCloudPlatform `nfs-server-docker` image for running an NFSv3-compatible server inside a container. The image is based on Debian 11 and packages the upstream `nfs-kernel-server` along with the original entrypoint script that manages exports.

## Image contents

-   Base image: `debian:11-slim`
-   NFS server: `nfs-kernel-server` pinned to `1:1.3.4-6` (exported as `C2D_RELEASE=1.3.4`)
-   Entrypoint: `docker-entrypoint.sh`, which configures `/etc/exports`, optional group ownership, and starts `rpcbind`, `rpc.mountd`, and `rpc.nfsd`
-   Default export directory: `/exports` (declared as a volume)
-   Ports exposed: `2049/tcp` (NFS) and `20048/tcp` (mountd). Runtime usage typically also requires UDP 2049 and TCP/UDP 111.

## Published image

GitHub Actions builds and pushes the image to GitHub Container Registry on every push to `main` or when manually triggered via the **Build and Push NFS Server Image** workflow.

-   Registry: `ghcr.io`
-   Image: `ghcr.io/cloudx-labs/gcp-nfs-server`
-   Tags: `latest`, branch names, semantic versions (when applicable), and commit SHAs generated via `docker/metadata-action`

To pull the latest build:

```
docker pull ghcr.io/cloudx-labs/gcp-nfs-server:latest
```

## Running the NFS server container

### Minimal example

```
docker run -d \
  --name nfs-server \
  --privileged \
  --network host \
  -v /srv/nfs/data:/exports \
  ghcr.io/cloudx-labs/gcp-nfs-server:latest
```

-   `--privileged` (or equivalent capabilities) is required so the kernel NFS daemons can start.
-   `--network host` simplifies exposing the required NFS/RPC ports. If host networking is not an option, publish TCP/UDP ports `111`, `2049`, and `20048` explicitly.
-   Mount one or more host directories at `/exports` (or other paths you pass as container arguments).

### Exporting multiple paths

Pass each export path to the container. The entrypoint writes them to `/etc/exports` with the options `*(rw,fsid=0,sync,no_subtree_check,no_root_squash)`:

```
docker run -d --privileged --network host \
  -v /srv/nfs/data:/exports/data \
  -v /srv/nfs/backups:/exports/backups \
  ghcr.io/cloudx-labs/gcp-nfs-server:latest \
  /exports/data /exports/backups
```

### Setting the export group

Use `-G <gid>` to reassign group ownership of the exported directories inside the container before starting NFS:

```
docker run -d --privileged --network host \
  -v /srv/nfs/data:/exports \
  ghcr.io/cloudx-labs/gcp-nfs-server:latest \
  -G 2000 /exports
```

Make sure the directories exist on the host with the desired permissions prior to starting the container.

### Mounting from a client

From another machine, mount the export as usual (replace `<server>` with the host running the container):

```
sudo mount -t nfs -o vers=3 <server>:/exports /mnt/nfs
```

## Building locally

```
git clone https://github.com/cloudx-labs/gcp-nfs-server.git
cd gcp-nfs-server
docker build -t gcp-nfs-server:local .
```

Publish your local build to a registry of choice or run it directly with the commands above.

## Contributing

Issues and pull requests are welcome. When contributing changes that affect the image contents, update this README and verify the container starts successfully by running it locally.

## License

This repository is distributed under the MIT License (see `LICENSE`). Portions derived from the original GoogleCloudPlatform `nfs-server-docker` project retain their respective notices.
