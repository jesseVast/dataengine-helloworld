# Create Pipeline

Wire the registered `helloworld` function to an S3 element trigger so every new object under a prefix runs the function. Commands follow the [VAST DataEngine](https://docs.vastdata.com/develop/docs/dev-dataengine) 5.5 guides for [triggers](https://docs.vastdata.com/develop/docs/dev-dataengine-guides-triggers) and [pipelines as code](https://docs.vastdata.com/develop/docs/dev-dataengine-guides-pipelines-as-code).

## What a pipeline is

A **pipeline** is the unit DataEngine deploys to Kubernetes. It is a named set of:

- **Triggers** — sources of events (here, one S3 element trigger)
- **Function deployments** — which functions to run (here, `helloworld`)
- **Links** — which events go to which functions (trigger → function)

Creating the pipeline only saves a draft; **deploy** pulls the function image, starts the runtime, and connects it to the trigger's topic. Until status is `Ready`, nothing runs on uploads.

Triggers and functions are reusable objects of their own. The pipeline does not own the trigger or the image; it only references them by VRN and wires them together. See [Concepts](https://docs.vastdata.com/develop/docs/dev-dataengine-concepts-index).

## The test trigger: helloworld-uploads

This guide uses one **element trigger** named `helloworld-uploads`:

| Setting | Value | Why |
| --- | --- | --- |
| Type | Element | Watches a bucket (view) for object changes |
| Source bucket | `$BUCKET` | The view your platform team set up for triggers |
| Event | `ObjectCreated:*` | Fire when an object is created (Put, Copy, and similar) |
| Name prefix | `helloworld/` | Only keys under that prefix; other uploads on the same bucket are ignored |
| Topic | Tenant default | No broker/topic flags → events go to the tenant's default topic |

It is a smoke-test trigger: upload something under `helloworld/`, get one CloudEvent, confirm the function ran. Filtering in the trigger (the prefix) is cheaper than filtering in the handler, and it keeps this demo from reacting to every object in the bucket.

When an object is created, the cluster delivers a CloudEvent of type `vastdata.com:Element.ElementCreated` (not the S3-style `ObjectCreated:*` name you configured). The runtime wraps that for your handler; `event.bucket` and `event.object_key` come from the event's `elementpath`.

## How it works

```text
S3 upload to s3://$BUCKET/helloworld/...
        │
        ▼
 helloworld-uploads (element trigger)
        │  publishes CloudEvent to default topic
        ▼
 helloworld pipeline (link: uploads → helloworld)
        │
        ▼
 helloworld function (init once, handler per event)
        │
        ▼
 logs: Initialized... / Handler ... / Function returned: Hello World
```

1. You upload an object whose key starts with `helloworld/`.
2. The element trigger matches (`ObjectCreated:*` + prefix) and publishes one event to the tenant's default topic.
3. The deployed pipeline's link sends that event to the `helloworld` function deployment.
4. The function instance runs `handler(ctx, event)`, logs the event, and returns `"Hello World"`.
5. You read the result with `vastde logs get helloworld`.

A local run ([TESTING.md](TESTING.md)) skips steps 2–3: you send the event yourself. This pipeline article is the path that uses a real bucket trigger on the cluster.

## Prerequisites

Complete [README.md](README.md) through **Push** and **Optional: register the function**. You need:

- A published `helloworld` function (`vastde functions get helloworld`)
- A source bucket configured for element triggers (platform team; see [Setup prerequisites](https://docs.vastdata.com/develop/docs/dev-dataengine-operations-setup-prerequisites))
- An S3 access key that can write to that bucket
- `vastde` configured for your tenant

Set these for the rest of the article:

```bash
export BUCKET="<source bucket>"
export S3_ENDPOINT="http://<S3 address>"
export AWS_ACCESS_KEY_ID="<access key>"
export AWS_SECRET_ACCESS_KEY="<secret key>"
```

## 1. Create the S3 element trigger

An element trigger watches one bucket and publishes one event per matching change. This one fires for every object created under `helloworld/`:

```bash
vastde triggers create element \
  --name helloworld-uploads \
  --description "New objects under helloworld/ in the source bucket" \
  --source-bucket "$BUCKET" \
  --event "ObjectCreated:*" \
  --name-prefix helloworld/
```

Check it:

```bash
vastde triggers get helloworld-uploads
```

Expect `Type: Element`, `Status: Ready`, and `SourceBucket` equal to `$BUCKET`. With no broker or topic flags, the trigger uses the tenant's default topic.

Useful flags:

| Flag | Purpose |
| --- | --- |
| `--event "ObjectCreated:*"` | Fire on object create (required; repeat for more events) |
| `--name-prefix helloworld/` | Only keys that start with this string |
| `--name-suffix` | Optional key suffix filter |
| `--source-types S3` | Optional; defaults cover S3 uploads on a typical lab |

Accepted create events include `ObjectCreated:*`, `ObjectRemoved:*`, `ObjectTagging:Put`, and `ObjectTagging:Delete`. The handler sees `vastdata.com:Element.ElementCreated` for creates, not the S3-style name. See [Triggers](https://docs.vastdata.com/develop/docs/dev-dataengine-guides-triggers).

## 2. Find the compute cluster and namespace

A pipeline is fixed to one cluster and one namespace at create time:

```bash
vastde compute-clusters list
vastde compute-clusters get <cluster> -o yaml
```

From the YAML, copy:

- `vrn` — for example `vast:dataengine:kubernetes-clusters:<cluster>`
- Your tenant namespace under `namespaces` — on a VAST compute cluster, usually `dataengine-ns-<tenant GUID>`

## 3. Write the pipeline file

[pipeline.yaml](pipeline.yaml) in this repo is the template. Fill in the cluster VRN and namespace:

```yaml
name: helloworld
description: Run helloworld on every new object under helloworld/ in the source bucket
kubernetes_cluster_vrn: vast:dataengine:kubernetes-clusters:<cluster>
namespace: <namespace>
manifest:
  triggers:
    - name: uploads
      vrn: vast:dataengine:triggers:helloworld-uploads
  function_deployments:
    - name: helloworld
      function_vrn: vast:dataengine:functions:helloworld
  links:
    - source: [uploads]
      destination: [helloworld]
```

The manifest gives the trigger and function local names (`uploads`, `helloworld`) and links them: every event from `uploads` goes to `helloworld`. The function and trigger must already exist; the pipeline only references their VRNs.

## 4. Create and deploy the pipeline

```bash
vastde pipelines create --config @pipeline.yaml --deploy
```

Poll until the status is `Ready` (or inspect `Error`):

```bash
vastde pipelines get helloworld
```

If create fails, the file usually names a missing function, trigger, link, or namespace. If create succeeds but deploy fails, the pipeline is left as `Draft` — fix the file, then `vastde pipelines update helloworld --config @pipeline.yaml` and `vastde pipelines deploy helloworld`, or delete and create again. See [Pipelines as code](https://docs.vastdata.com/develop/docs/dev-dataengine-guides-pipelines-as-code).

## 5. Upload an object

Upload any small file under the trigger prefix (AWS CLI, as in the DataEngine quickstart):

```bash
echo "hello" > /tmp/hello.txt
aws --region us-east-1 --endpoint-url "$S3_ENDPOINT" \
  s3 cp /tmp/hello.txt "s3://$BUCKET/helloworld/hello.txt"
```

Prefer **s3cmd**? Install, point it at the VAST S3 VIP, and `put` under `helloworld/`: [RECOMMENDATIONS.md](RECOMMENDATIONS.md#put-objects-with-s3cmd).

## 6. Read the logs

```bash
vastde logs get helloworld --since 10m --scope user
```

You should see `Initialized...` (from `init`, once per instance) and a handler line for the upload. With default scope (or without `--scope user`), the runtime also logs `Function returned: Hello World`.

Traces:

```bash
vastde traces list helloworld --since 10m
```

See [Observability](https://docs.vastdata.com/develop/docs/dev-dataengine-guides-observability).

## Clean up

Delete the pipeline first (stops the function on the compute cluster), then the trigger. Keep or delete the function and registry image as you prefer:

```bash
vastde pipelines delete helloworld
vastde triggers delete helloworld-uploads
# optional:
# vastde functions delete helloworld
aws --region us-east-1 --endpoint-url "$S3_ENDPOINT" \
  s3 rm "s3://$BUCKET/helloworld/hello.txt"
```

## See also

- [README.md](README.md) — install `vastde`, build, push, register
- [TESTING.md](TESTING.md) — local run before push
- [RECOMMENDATIONS.md](RECOMMENDATIONS.md) — s3cmd uploads and smoke-test tips
- [Triggers](https://docs.vastdata.com/develop/docs/dev-dataengine-guides-triggers)
- [Pipelines as code](https://docs.vastdata.com/develop/docs/dev-dataengine-guides-pipelines-as-code)
- [Pipeline YAML reference](https://docs.vastdata.com/develop/docs/dev-dataengine-reference-pipeline-yaml)
- [Quickstart](https://docs.vastdata.com/develop/docs/dev-dataengine-quickstart)
