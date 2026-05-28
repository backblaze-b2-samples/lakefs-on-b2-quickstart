# lakeFS on Backblaze B2: quickstart

A self-contained Docker Compose stack that runs [lakeFS](https://lakefs.io) with [Backblaze B2](https://www.backblaze.com/cloud-storage) as the blockstore. Lets you create a versioned data lake on top of B2 in about five minutes, with branching, commits, and rollback over an existing B2 bucket.

The stack runs two containers:

* `treeverse/lakefs:1.34` on `http://localhost:8000` for the web UI and the API that `lakectl`, the lakeFS Python SDK, and the [companion notebook](https://github.com/backblaze-b2-samples/notebooks/tree/main/lakefs-b2-dataset-versioning) talk to.
* `postgres:14` as the metadata KV store (repos, branches, commits, refs).

The actual data objects land in your B2 bucket via lakeFS's S3-compatible blockstore adapter. The recently-merged [treeverse/lakeFS#10426](https://github.com/treeverse/lakeFS/pull/10426) tags every outbound S3 request with a `lakefs/<version>` `User-Agent` so you can attribute the traffic on the B2 side.

## Contents

* [Prerequisites](#prerequisites)
* [Quick start](#quick-start)
* [Five-minute walkthrough](#five-minute-walkthrough)
* [Cleanup](#cleanup)
* [Troubleshooting](#troubleshooting)
* [What this sample is not](#what-this-sample-is-not)

## Prerequisites

1. **Docker Engine** 20.10+ with the Compose v2 plugin (`docker compose`, not `docker-compose`). Verify with `docker compose version`.
2. **A Backblaze B2 account** with one empty bucket dedicated to lakeFS data. Sign up at [backblaze.com/cloud-storage](https://www.backblaze.com/cloud-storage). The free tier (10 GB) is plenty for the walkthrough.
3. **A B2 Application Key** scoped to the bucket above. In the B2 console, go to **Application Keys** &rarr; **Add a New Application Key**, restrict it to the bucket you just created, and grant `listBuckets`, `listFiles`, `readFiles`, `writeFiles`, and `deleteFiles`. Save the `keyID` (an `AWS_ACCESS_KEY_ID`-shaped string) and `applicationKey` (an `AWS_SECRET_ACCESS_KEY`-shaped string); the `applicationKey` is shown only once.
4. **The bucket's region.** Visible on the bucket detail page as `Endpoint: s3.<region>.backblazeb2.com`. Typical values: `us-west-001`, `us-west-002`, `us-west-004`, `us-east-005`, `eu-central-003`. Both the endpoint URL and the region string need to match exactly.
5. **(Optional) `lakectl`**, the lakeFS CLI. Installation: `brew install lakefs` on macOS, or grab a binary from [the releases page](https://github.com/treeverse/lakeFS/releases). The walkthrough below shows both `lakectl` and pure-curl commands; pick one.

## Quick start

```bash
# 1. Clone this sample.
git clone https://github.com/backblaze-b2-samples/lakefs-on-b2-quickstart.git
cd lakefs-on-b2-quickstart

# 2. Copy the env template and fill in your B2 details.
cp .env.example .env
$EDITOR .env

# 3. Start the stack.
docker compose up -d

# 4. Wait for lakeFS to report healthy.
docker compose ps
```

When `docker compose ps` shows `lakefs-on-b2-quickstart-lakefs-1` with status `Up (healthy)`, lakeFS is ready. Open `http://localhost:8000` in a browser. Log in with the `LAKEFS_ADMIN_ACCESS_KEY_ID` and `LAKEFS_ADMIN_SECRET_ACCESS_KEY` values you set in `.env`.

## Five-minute walkthrough

The walkthrough creates a repo, makes two changes on a branch, diffs the branch against `main`, merges, then rolls back. Everything below assumes you have `lakectl` installed and configured with the same credentials as in `.env`:

```bash
# One-time lakectl config; writes ~/.lakectl.yaml.
lakectl config
# Server endpoint URL: http://localhost:8000
# Access key ID:       <LAKEFS_ADMIN_ACCESS_KEY_ID from .env>
# Secret access key:   <LAKEFS_ADMIN_SECRET_ACCESS_KEY from .env>
```

### 1. Create a repo backed by your B2 bucket

```bash
lakectl repo create \
    lakefs://quickstart \
    s3://${B2_LAKEFS_DATA_BUCKET}/quickstart \
    --default-branch main
```

This registers a new lakeFS repo named `quickstart` whose storage namespace is `s3://${B2_LAKEFS_DATA_BUCKET}/quickstart` (the `s3://` scheme is from lakeFS's perspective; the actual bytes go to B2 via the configured endpoint).

### 2. Upload an object on `main`

```bash
echo "version 1" > /tmp/hello.txt
lakectl fs upload \
    -s /tmp/hello.txt \
    lakefs://quickstart/main/hello.txt
lakectl commit lakefs://quickstart/main -m "initial hello"
```

### 3. Branch, modify, commit

```bash
lakectl branch create \
    lakefs://quickstart/experiment \
    --source lakefs://quickstart/main

echo "version 2" > /tmp/hello.txt
lakectl fs upload \
    -s /tmp/hello.txt \
    lakefs://quickstart/experiment/hello.txt
lakectl commit lakefs://quickstart/experiment -m "update hello"
```

### 4. Diff the branch against `main`

```bash
lakectl diff lakefs://quickstart/main lakefs://quickstart/experiment
# + modified hello.txt
```

### 5. Merge `experiment` into `main`

```bash
lakectl merge \
    lakefs://quickstart/experiment \
    lakefs://quickstart/main
```

### 6. Roll back

```bash
# View commit history.
lakectl log lakefs://quickstart/main

# Reset main to the commit before the merge.
lakectl branch reset lakefs://quickstart/main \
    --commit <commit-id-from-log-output>
```

The data in B2 is unchanged across all of the above; lakeFS rewrites references, not bytes. Confirm by listing the underlying B2 bucket with the AWS CLI:

```bash
aws s3 ls "s3://${B2_LAKEFS_DATA_BUCKET}/quickstart/" \
    --endpoint-url "${AWS_ENDPOINT_URL}"
```

You should see lakeFS's internal layout (`_lakefs/`, content-addressed object names under hashed prefixes). Do not edit these directly; lakeFS manages the layout.

## Cleanup

```bash
docker compose down            # stop containers, keep the Postgres volume
docker compose down -v         # stop containers and drop the Postgres volume
```

`docker compose down -v` wipes all lakeFS metadata (repos, branches, commits) but **does not** touch your B2 bucket. The committed objects remain in B2 under the lakeFS storage namespace until you delete them yourself. To start completely fresh, also empty the B2 bucket (the B2 console has a "Delete All Files" action under the bucket settings).

## Troubleshooting

### `403 Forbidden` from lakeFS on startup, with B2 error in logs

Check `docker compose logs lakefs` for a line like `error creating object: ... 403`. Most common causes:

1. **`LAKEFS_BLOCKSTORE_S3_FORCE_PATH_STYLE` is not `true`.** B2 requires path-style addressing (`https://s3.<region>.backblazeb2.com/<bucket>/<key>`). Virtual-hosted style (`https://<bucket>.s3.<region>.backblazeb2.com/<key>`) fails on B2. This sample sets `force_path_style: true` already; verify your environment didn't override it.
2. **Region mismatch between endpoint and bucket.** If `AWS_ENDPOINT_URL=https://s3.us-west-004.backblazeb2.com` but the bucket lives in `us-west-001`, B2 returns the misleading error `InvalidAccessKeyId`. Confirm the bucket's region in the B2 console and update both `AWS_ENDPOINT_URL` and `AWS_DEFAULT_REGION` to match.
3. **Application Key scope.** The key must grant `writeFiles` on the target bucket. A "read-only" key fails when lakeFS tries to write the initial repo metadata.

### `port is already allocated` on `docker compose up`

Another process is bound to `8000` or `5432` on your host. Change the lakeFS port in `docker-compose.yml`:

```yaml
    ports:
      - "8001:8000"   # use 8001 on the host instead
```

Postgres is not published to the host by default in this sample (containers reach it on the internal Compose network), so a host conflict on 5432 will not occur unless you also added a port mapping.

### lakeFS UI says "first-time setup" rather than letting me log in

The `LAKEFS_INSTALLATION_*` env vars are evaluated only on the very first lakeFS startup against an empty Postgres database. If you previously started lakeFS without those vars set, the UI runs its interactive setup wizard. Two fixes:

* Run the setup wizard once, choose your own admin credentials, and skip the env-var path.
* Or: `docker compose down -v` to drop the Postgres volume, then `docker compose up -d` again so lakeFS sees an empty DB on first start and honors the `LAKEFS_INSTALLATION_*` values.

### Verifying that B2 sees a `lakefs/<version>` `User-Agent`

The lakeFS S3 block adapter sets `User-Agent: aws-sdk-go-v2/... lakefs/<version> ...` on every outbound request (introduced in [PR #10426](https://github.com/treeverse/lakeFS/pull/10426), shipped from lakeFS 1.34). Confirm with `tcpdump` against the lakeFS container, or look at your B2 server-side access logs if you have them enabled.

## What this sample is not

* **Not production-ready.** The Postgres password, the lakeFS auth encryption secret, and the admin credentials are all literal example values from `.env.example`. Rotate them before exposing the API to anything other than localhost.
* **Not high-availability.** Single Postgres, single lakeFS, single-node Compose. For production, use a managed Postgres and run lakeFS behind a load balancer (or sign up for [lakeFS Cloud](https://lakefs.cloud)).
* **Not a B2 native-API demo.** lakeFS talks to B2 via the S3-compatible API. lakeFS does not currently support B2's native (non-S3) API.

## Related

* lakeFS docs: <https://docs.lakefs.io>
* lakeFS source: <https://github.com/treeverse/lakeFS>
* Companion notebook: [`backblaze-b2-samples/notebooks/lakefs-b2-dataset-versioning`](https://github.com/backblaze-b2-samples/notebooks/tree/main/lakefs-b2-dataset-versioning) walks through branch-per-experiment dataset versioning for ML, using the stack from this repo as the lakeFS backend.
* Backblaze B2 S3-compatible API: <https://www.backblaze.com/docs/cloud-storage-s3-compatible-api>

## License

[MIT](LICENSE).
