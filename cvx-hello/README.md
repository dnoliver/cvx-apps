# cvx-hello

The "hello world" of Converix apps. If you're new to the platform and just want
to see something deploy, run, and answer HTTP traffic, this is the chart to
start with.

## What it does

`cvx-hello` deploys an [nginx](https://nginx.org/) web server that serves its
default welcome page over HTTP. It's deliberately boring:

- One Deployment running the `nginx:1.31.3` container image.- One LoadBalancer
  Service exposing port `80`, labelled with `networking.converix.io/vlan: pax`
  so Converix knows how to publish it.
- A NetworkPolicy that adapts to the Service type: with `LoadBalancer` (the
  default) it allows traffic from anywhere — every namespace in the cluster and
  clients outside it via `0.0.0.0/0`; with `ClusterIP` it allows only in-cluster
  traffic from every namespace.
- Liveness and readiness probes (HTTP GET on `/`) enabled by default.

There is no app logic of its own. The point is to be the simplest possible
example of a Converix app — a known-good starting point you can clone, modify,
and break things against while learning.

## What's in here

```text
cvx-hello/
├── Makefile              # builds the offline bundle (image .tar + chart OCI archive)
├── cvx-hello/            # the Helm chart
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

The fastest way to see `cvx-hello` running is to build the offline bundle and
hand it to your Converix environment:

```sh
cd cvx-hello
make
```

That produces exactly two deliverables under `dist/`:

- a Docker image archive (`.tar`) in **hybrid format** — an OCI layout
  (`index.json`, `oci-layout`, `blobs/`) plus Docker's legacy `manifest.json` in
  the same tar. Requires the containerd image store (Docker Desktop: Settings →
  General → "Use containerd for pulling and storing images"); the Makefile fails
  if the archive is missing either half.
- the chart as an OCI artifact (`cvx-hello-<version>.tgz`) — a gzipped
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

The fastest path from "hello world" to "my first real Converix app" is to copy
this chart and tweak it:

1. Copy the `cvx-hello/` directory to a new name, e.g. `cvx-myapp/`.
2. In `Chart.yaml`, change `name` (and `appVersion` if you want to track it).
3. In `values.yaml`, change `image.repository` and `image.tag` to your image.
4. Pick the right Converix `vlan` for your Service (`crew`, `seatbox`, or `pax`
   — these map to different audiences on the platform).
5. Run `make` to repackage and ship.

That's it — probes come enabled by default, and everything else (autoscaling,
ingress, Gateway API HTTPRoute) is opt-in via `values.yaml` and ready to use
when you need it.

## Learn more

- [Helm](https://helm.sh/) — for how charts work
- [Kubernetes Services](https://kubernetes.io/docs/concepts/services-networking/service/)
  — for what `LoadBalancer` actually means
- [Kubernetes NetworkPolicies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
  — for why there's a NetworkPolicy in a "hello world"
