# cvx-nodered

[Node-RED](https://nodered.org/) — the flow-based programming tool for wiring
together APIs, devices and services — packaged as a Converix app.

## What it does

`cvx-nodered` deploys a Node-RED editor and runtime:

- One Deployment running the `nodered/node-red:5.0.1-minimal` container image.
- One LoadBalancer Service exposing port `1880` (Node-RED's editor and HTTP
  endpoints), labelled with `networking.converix.io/vlan: pax` so Converix knows
  how to publish it.
- A NetworkPolicy that adapts to the Service type: with `LoadBalancer` (the
  default) it allows traffic from anywhere — every namespace in the cluster and
  clients outside it via `0.0.0.0/0`; with `ClusterIP` it allows only in-cluster
  traffic from every namespace. (For this app, a recent commit additionally
  allows all *egress* — Node-RED flows typically need to call out to external
  APIs, MQTT brokers and the like.)
- Liveness and readiness probes (HTTP GET on `/`) enabled by default.

Open the editor in a browser and start wiring flows. Note that the chart does
not yet provision persistent storage, so installed nodes and deployed flows are
lost when the pod restarts — treat it as a demo-grade deployment.

## What's in here

```text
cvx-nodered/
├── Makefile              # builds the offline bundle (image .tar + chart OCI archive)
├── cvx-nodered/          # the Helm chart
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

The fastest way to see `cvx-nodered` running is to build the offline bundle and
hand it to your Converix environment:

```sh
cd cvx-nodered
make
```

That produces exactly two deliverables under `dist/`:

- a Docker image archive (`.tar`) in **hybrid format** — an OCI layout
  (`index.json`, `oci-layout`, `blobs/`) plus Docker's legacy `manifest.json` in
  the same tar. Requires the containerd image store (Docker Desktop: Settings →
  General → "Use containerd for pulling and storing images"); the Makefile fails
  if the archive is missing either half.
- the chart as an OCI artifact (`cvx-nodered-<version>.tgz`) — a gzipped
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

1. Copy the `cvx-nodered/` directory to a new name, e.g. `cvx-myapp/`.
2. In `Chart.yaml`, change `name` (and `appVersion` if you want to track it).
3. In `values.yaml`, change `image.repository` and `image.tag` to your image.
4. Pick the right Converix `vlan` for your Service (`crew`, `seatbox`, or `pax`
   — these map to different audiences on the platform).
5. Run `make` to repackage and ship.

## Learn more

- [Node-RED](https://nodered.org/docs/) — the runtime and its docs
- [Kubernetes Services](https://kubernetes.io/docs/concepts/services-networking/service/)
  — for what `LoadBalancer` actually means
- [Kubernetes NetworkPolicies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
  — for why there's a NetworkPolicy in the chart
