# Recommendations

Practical tips for running this helloworld smoke test against a VAST cluster. Prefer these over improvising paths that fight DataEngine defaults.

## Put objects with s3cmd

[PIPELINE.md](PIPELINE.md) shows uploads with the AWS CLI. [s3cmd](https://s3tools.org/s3cmd) is a good alternative for putting objects that fire the element trigger. VAST documents install and config in [Install and configure s3cmd](https://kb.vastdata.com/documentation/docs/install-and-configure-s3cmd).

### Install

```bash
# macOS
brew install s3cmd

# Debian / Ubuntu
sudo apt-get update && sudo apt-get install -y s3cmd

# RHEL / CentOS / Rocky
sudo yum install s3cmd -y

# Or via pip
pip install s3cmd
```

### Configure for a VAST S3 VIP

Create a config file (for example `~/.s3cfg-vast` or `vast.s3cfg` in the project — keep secrets out of Git):

```ini
[default]
access_key = <ACCESS-KEY>
secret_key = <SECRET-KEY>
host_base = <CLUSTER VIRTUAL IP>
host_bucket = <CLUSTER VIRTUAL IP>
use_https = false
```

VAST notes for this file:

- `host_base` and `host_bucket` are both the cluster S3 VIP (path-style). Do not leave `host_bucket` as an Amazon-style `%(bucket)s.s3.amazonaws.com` default.
- Do **not** put a port on the VIP. Specifying a port (including the default) can break signature v4.
- Set `use_https = true` only when the cluster has an S3 SSL cert and you intend HTTPS (port 443). Otherwise use `false` for HTTP.

Interactive setup also works: `s3cmd --configure`, then edit the file so both host fields are the VIP.

### Put an object under the trigger prefix

The helloworld trigger watches `helloworld/`. Put under that prefix or the pipeline will not run:

```bash
export BUCKET="<source bucket>"

echo "hello" > /tmp/hello.txt
s3cmd -c ~/.s3cfg-vast put /tmp/hello.txt "s3://$BUCKET/helloworld/hello.txt"
```

List and remove:

```bash
s3cmd -c ~/.s3cfg-vast ls "s3://$BUCKET/helloworld/"
s3cmd -c ~/.s3cfg-vast del "s3://$BUCKET/helloworld/hello.txt"
```

After a put, check `vastde logs get helloworld --since 10m` for `Function returned: Hello World` (see [PIPELINE.md](PIPELINE.md)).

Same credentials and bucket as the quickstart env vars (`BUCKET`, access key, secret key); s3cmd stores the endpoint in its config instead of `S3_ENDPOINT`.

## Other recommendations

| Recommendation | Why |
| --- | --- |
| Match `vastde` and the builder tag to the cluster (for example `v5.5.0-sp2`) | Mismatched CLI/builder images cause opaque build and deploy failures. Do not use `latest` or `dev` tags. |
| Run [TESTING.md](TESTING.md) before `docker push` | Catches import and handler errors on your machine before the registry or cluster sees the image. |
| `docker login <registry host>` before every push | Push uses local Docker credentials, not DataEngine registry settings. |
| New image tag per build (`v1`, `v2`, or a Git SHA); never `latest` | Each function revision records a tag; reusing a tag makes revisions ambiguous. |
| Wait until `vastde pipelines get helloworld` shows `Status: Ready` before uploading | Uploads during `InProgress` or `Error` may not run the handler you expect. |
| Keep a dedicated `--name-prefix` (here `helloworld/`) on the element trigger | Filters at the trigger so other objects in the same bucket do not invoke this pipeline. |
| Delete the pipeline before deleting the trigger | The pipeline references the trigger VRN; remove the consumer first. |
| Keep `~/.s3cfg*`, AWS keys, and `config.yaml` secrets out of Git | Credentials for upload and localrun do not belong in the repo. |
| Confirm the source view is provisioned for element triggers | Application users cannot create trigger source views; ask the platform team if `Bucket not found` or create fails. |

## See also

- [README.md](README.md) — install, build, push, register
- [TESTING.md](TESTING.md) — local run before push
- [PIPELINE.md](PIPELINE.md) — S3 trigger, pipeline, upload, logs
- [VAST: Install and configure s3cmd](https://kb.vastdata.com/documentation/docs/install-and-configure-s3cmd)
- [Setup prerequisites](https://docs.vastdata.com/develop/docs/dev-dataengine-operations-setup-prerequisites)
