# cvx-redis

[Redis](https://redis.io/) — the in-memory data store for caching, messaging and
vector search — packaged as a Converix app.

## What it does

`cvx-redis` deploys a single Redis server:

- One Deployment running the `redis:8.8.0-trixie` container image, plus a second
  Deployment (`deployment-consumer.yaml`) for a Redis consumer pod — see the
  templates for what the consumer does in your setup.
- One **ClusterIP** Service on port `6379`, labelled with
  `networking.converix.io/vlan: pax`. Unlike the web apps in this repo, Redis is
  not published outside the cluster — it's a backing service meant to be reached
  by other in-cluster apps at `<fullname>:6379`.
- A NetworkPolicy that adapts to the Service type: with `ClusterIP` (the default
  here) it allows only in-cluster traffic from every namespace; with
  `LoadBalancer` it would also allow clients outside the cluster via
  `0.0.0.0/0`.
- Redis-appropriate probes instead of the usual HTTP ones: liveness on a TCP
  socket to port `redis`, readiness via `redis-cli ping`.

Being in-memory only, data is lost when the pod restarts — the chart does not
yet provision persistent storage. Treat it as a scratch cache or a demo-grade
deployment.

## What's in here

```text
cvx-redis/
├── Makefile              # builds the offline bundle (image .tar + chart OCI archive)
├── cvx-redis/            # the Helm chart
│   ├── Chart.yaml
│   ├── values.yaml
│   ├── templates/        # incl. templates/tests/ (helm test connection check)
│   └── charts/           # empty; subcharts would go here
├── dist/                 # the two deliverables (image .tar + chart .tgz)
└── build/                # intermediates (chart .tgz, OCI layout, unpack copies)
```

The chart is a standard [Helm](https://helm.sh/) chart for Kubernetes, with a
few Converix-specific touches (the `vlan` service label and the NetworkPolicy).
See the [top-level repo README](../README.md) for a tour of the shared chart
features used here.

## Try it

The fastest way to get `cvx-redis` running is to build the offline bundle and
hand it to your Converix environment:

```sh
cd cvx-redis
make
```

That produces exactly two deliverables under `dist/`:

- a Docker image archive (`.tar`) in **hybrid format** — an OCI layout
  (`index.json`, `oci-layout`, `blobs/`) plus Docker's legacy `manifest.json` in
  the same tar. Requires the containerd image store (Docker Desktop: Settings →
  General → "Use containerd for pulling and storing images"); the Makefile fails
  if the archive is missing either half.
- the chart as an OCI artifact (`cvx-redis-<version>.tgz`) — a gzipped
  `oci-archive` of the packaged chart, with Helm's media types and content
  digests intact.

To clean the generated folders:

```sh
make clean    # remove dist/ and build/
```

**Requirements:** [Docker](https://www.docker.com/) (containerd image store
enabled), [Helm](https://helm.sh/), [ORAS](https://oras.land/) and
[`yq`](https://github.com/mikefarah/yq). Optionally
[Skopeo](https://github.com/containers/skopeo) for `make inspect`.

## Make it your own

1. Copy the `cvx-redis/` directory to a new name, e.g. `cvx-myapp/`.
2. In `Chart.yaml`, change `name` (and `appVersion` if you want to track it).
3. In `values.yaml`, change `image.repository` and `image.tag` to your image.
4. Pick the right Service exposure: keep `ClusterIP` for a backing service, or
   switch to `LoadBalancer` and choose a Converix `vlan` (`crew`, `seatbox`, or
   `pax`) to publish it.
5. Run `make` to repackage and ship.

## Learn more

- [Redis](https://redis.io/docs/) — the server and its docs
- [Kubernetes Services](https://kubernetes.io/docs/concepts/services-networking/service/)
  — for the difference between `ClusterIP` and `LoadBalancer`
- [Kubernetes NetworkPolicies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
  — for why there's a NetworkPolicy in the chart
