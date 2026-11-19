# DataEngine-helloworld

A minimal [VAST DataEngine](https://docs.vastdata.com/develop/docs/dev-dataengine) function you can build and push to confirm a deployment path. The handler logs each event and returns `"Hello World"` (visible in the function logs as `Function returned: Hello World`).

## Before you start

- [Docker](https://docs.docker.com/get-docker/) running on your machine
- DataEngine enabled on your tenant, with a compute cluster, namespace, and container registry already linked (see [Setup prerequisites](https://docs.vastdata.com/develop/docs/dev-dataengine-operations-setup-prerequisites))
- Push credentials for that registry

## Install vastde

The CLI lives in [vast-data/dataengine-cli](https://github.com/vast-data/dataengine-cli). Use the release your platform team names for the cluster. The latest published release is on [releases/latest](https://github.com/vast-data/dataengine-cli/releases/latest); the 5.5 docs use [v5.5.0-sp2](https://github.com/vast-data/dataengine-cli/releases/tag/v5.5.0-sp2). Prefer a release tag over a `dev` build from `main`.

### macOS (Apple Silicon)

```bash
curl -fsSL -o vastde https://github.com/vast-data/dataengine-cli/releases/download/v5.5.0-sp2/vastde_darwin_arm64
chmod +x vastde
sudo mv vastde /usr/local/bin
vastde version
```

### macOS (Intel)

```bash
curl -fsSL -o vastde https://github.com/vast-data/dataengine-cli/releases/download/v5.5.0-sp2/vastde_darwin_amd64
chmod +x vastde
sudo mv vastde /usr/local/bin
vastde version
```

### Linux (x86_64)

```bash
curl -fsSL -o vastde https://github.com/vast-data/dataengine-cli/releases/download/v5.5.0-sp2/vastde_linux_amd64
chmod +x vastde
sudo mv vastde /usr/local/bin
vastde version
```

### Linux (ARM64)

```bash
curl -fsSL -o vastde https://github.com/vast-data/dataengine-cli/releases/download/v5.5.0-sp2/vastde_linux_arm64
chmod +x vastde
sudo mv vastde /usr/local/bin
vastde version
```

### Windows (x86_64)

Download [vastde_windows_amd64.exe](https://github.com/vast-data/dataengine-cli/releases/download/v5.5.0-sp2/vastde_windows_amd64.exe) and add it to your `PATH`.

## Configure the CLI

In `vastde` v5.5.0-sp2, `config init` with flags exits and writes nothing until a config file exists. Create the file first, then set your tenant:

```bash
vastde config init
vastde config init --vms-url https://<vms-address> --tenant <tenant> \
  --username <user> --password <password> --force
vastde builders set v5.5.0-sp2
vastde config view
```

Pin the builder tag to the same release as the CLI. Do not use `latest`.

## Build

From the repository root, build the image locally (do not push yet):

```bash
export REGISTRY="<registry host>"
export IMAGE_TAG="v1"

vastde functions build "$REGISTRY/helloworld" \
  --target ./helloworld \
  --image-tag "$IMAGE_TAG"
```

This tags the image as `<registry host>/helloworld:<tag>` on your machine. Use a new tag for every build (a version or Git commit), never `latest`. The build pulls the builder and base image from Docker Hub and installs packages from PyPI, so the machine needs access to both (or mirrors).

See [Build and test locally](https://docs.vastdata.com/develop/docs/dev-dataengine-guides-build-and-test-locally).

## Test locally (recommended before push)

Before pushing to the registry, run the local smoke test in [TESTING.md](TESTING.md). Confirm `invoke` returns HTTP 204 and the container logs show `Function returned: Hello World`.

## Push

Log in to the designated registry first, then push the image you built:

```bash
docker login <registry host>

docker push "$REGISTRY/helloworld:$IMAGE_TAG"
```

`docker login` is required before `docker push`. `--push` on `vastde functions build` is not used here so you can test the local image first.

## Optional: register the function

After the image is in the registry, register it so DataEngine can pull it. `$REGISTRY_NAME` is the registry's name in DataEngine (`vastde container-registries list`), and it must be linked to the compute cluster and namespace your pipelines use:

```bash
export REGISTRY_NAME="<registry name>"

vastde functions create \
  --name helloworld \
  --description "Hello World smoke test" \
  --artifact-type image \
  --artifact-source helloworld \
  --image-tag "$IMAGE_TAG" \
  --container-registry "$REGISTRY_NAME" \
  --publish
```

## Next: S3 trigger and pipeline

After the function is registered, create an S3 element trigger and deploy a pipeline that runs `helloworld` on each upload: [PIPELINE.md](PIPELINE.md). Template: [pipeline.yaml](pipeline.yaml).

Tips for s3cmd uploads and other smoke-test habits: [RECOMMENDATIONS.md](RECOMMENDATIONS.md).

## Layout

```
.
├── README.md
├── TESTING.md
├── PIPELINE.md
├── RECOMMENDATIONS.md
├── pipeline.yaml
├── LICENSE
└── helloworld/
    ├── main.py           # init and handler
    ├── requirements.txt  # Python packages (empty for this demo)
    ├── Aptfile           # system packages
    └── customDeps        # local libraries (absolute paths)
```

## Links

- [VAST DataEngine docs](https://docs.vastdata.com/develop/docs/dev-dataengine)
- [Setup prerequisites](https://docs.vastdata.com/develop/docs/dev-dataengine-operations-setup-prerequisites)
- [Build and test locally](https://docs.vastdata.com/develop/docs/dev-dataengine-guides-build-and-test-locally)
- [Publish functions](https://docs.vastdata.com/develop/docs/dev-dataengine-guides-publish-functions)
- [Triggers](https://docs.vastdata.com/develop/docs/dev-dataengine-guides-triggers)
- [Pipelines as code](https://docs.vastdata.com/develop/docs/dev-dataengine-guides-pipelines-as-code)
- [Local testing](TESTING.md)
- [S3 trigger and pipeline](PIPELINE.md)
- [Recommendations](RECOMMENDATIONS.md)
- [dataengine-cli repository](https://github.com/vast-data/dataengine-cli)
- [Latest release](https://github.com/vast-data/dataengine-cli/releases/latest)
- [v5.5.0-sp2](https://github.com/vast-data/dataengine-cli/releases/tag/v5.5.0-sp2)
