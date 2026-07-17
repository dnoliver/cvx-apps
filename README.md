# cvx-apps

A collection of [Helm](https://helm.sh/) charts packaging applications for the
Converix platform. Each app is a self-contained chart plus a `Makefile` that
bundles the chart together with its container image into an offline-installable
artifact (`.tgz` + image `.tar`), so everything can be shipped into air-gapped
environments.

## Apps

| App                         | Description                                               | Image                                  | Service      | Port |
| --------------------------- | --------------------------------------------------------- | -------------------------------------- | ------------ | ---- |
| [cvx-2048](cvx-2048/)       | 2048 puzzle game                                          | `letsbootch/argocd-game2048-app:1.0.0` | LoadBalancer | 8080 |
| [cvx-gotify](cvx-gotify/)   | Simple server for sending and receiving push messages     | `gotify/server:2.9`                    | LoadBalancer | 80   |
| [cvx-hello](cvx-hello/)     | Hello World / nginx starter app                           | `nginx:1.31.3`                         | LoadBalancer | 80   |
| [cvx-nodered](cvx-nodered/) | Node-RED low-code flow programming for event-driven apps  | `nodered/node-red:5.0.1-minimal`       | LoadBalancer | 1880 |
| [cvx-redis](cvx-redis/)     | Redis data platform for caching, vector search, and NoSQL | `redis:8.8.0-trixie`                   | ClusterIP    | 6379 |

All charts are currently at chart version `0.2.0`.

## Repository layout

Every app follows the same structure:

```text
cvx-<app>/
├── Makefile              # builds the offline bundle (image .tar + chart .tgz)
├── cvx-<app>/            # the Helm chart
│   ├── Chart.yaml
│   ├── values.yaml
│   └── templates/        # deployment, service, ingress, httproute,
│                         # networkpolicy, hpa, serviceaccount, tests
└── dist/                 # build output (image .tar + packaged chart .tgz)
```

## Common chart features

All charts share a common template set, configured through `values.yaml`:

- **Deployment** with configurable `replicaCount`, image, probes, resources,
  security contexts, and node scheduling
  (`nodeSelector`/`tolerations`/`affinity`).
- **Service** — type and port are configurable. LoadBalancer services carry a
  `networking.converix.io/vlan` label (one of `crew`, `seatbox`, `pax`).
- **NetworkPolicy** (`networkPolicy.enabled`, on by default) whose behaviour
  adapts to the service type: `ClusterIP` allows traffic from all namespaces in
  the cluster; `LoadBalancer` also allows traffic from outside the cluster.
- **Ingress** (`ingress.enabled`, off by default).
- **Gateway API HTTPRoute** (`httpRoute.enabled`, off by default) — requires
  Gateway API resources and a suitable controller in the cluster.
- **HorizontalPodAutoscaler** (`autoscaling.enabled`, off by default).
- **ServiceAccount** and a Helm test (`templates/tests/`).

> `cvx-redis` additionally ships a consumer Deployment
> (`templates/deployment-consumer.yaml`) and uses a TCP liveness probe plus a
> `redis-cli ping` readiness probe instead of HTTP probes.

## Building an offline bundle

Each app's `Makefile` produces, under `dist/`, a Docker image archive pulled
from the chart's configured image and a packaged Helm chart.

```sh
cd cvx-hello
make            # build both the image .tar and the chart .tgz
make image      # only pull + save the image archive
make chart      # only lint, render, and package the chart
make clean      # remove dist/
```

Useful overrides:

```sh
make PLATFORM=linux/arm64      # target a different architecture (default: linux/amd64)
make CHART_DIR=path/to/chart   # override the auto-detected chart directory
```

The image to bundle is read from the chart's `values.yaml` (`image.repository` /
`image.tag`). The `chart` target runs `helm lint --strict` and `helm template`
before packaging.

**Requirements:** [Docker](https://www.docker.com/), [Helm](https://helm.sh/),
and [`yq`](https://github.com/mikefarah/yq).
