# Test locally

Run this after you build the image and **before** you push to the registry. A local run uses the same image and runtime as the cluster; it does not exercise triggers, retries, or cluster logging.

## Prerequisites

- Docker running
- Image already built locally (see [Build](README.md#build) in the root README)
- `REGISTRY` and `IMAGE_TAG` set to the same values used at build time

```bash
export REGISTRY="<registry host>"
export IMAGE_TAG="v1"
```

## 1. Start the function

```bash
vastde functions localrun "$REGISTRY/helloworld" \
  --image-tag "$IMAGE_TAG" \
  --port 18080 \
  --detach
```

Note the container ID printed at the end. Use it for logs and to stop the container later.

## 2. Send a test event

```bash
vastde functions invoke --generate-event --url http://localhost:18080/
```

Expect HTTP status **204**. That means the handler completed without raising.

## 3. Check the logs for Hello World

```bash
docker logs <container ID>
```

You should see lines like:

- `Initialized...` from `init`
- `Handler ...` from `handler`
- `Function returned: Hello World` — the handler return value is what appears in the logs

If `invoke` returns 500, or the logs show a traceback / missing `"Hello World"`, fix the code, rebuild, and repeat this test before pushing.

## 4. Stop the container

```bash
docker stop <container ID>
```

## Next step

When the local test passes, push the image: [Push](README.md#push).
