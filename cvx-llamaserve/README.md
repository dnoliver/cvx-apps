# cvx-llamaserve

A local LLM inference server, packaged as a Converix app — the
[llama.cpp](https://github.com/ggml-org/llama.cpp) server running an
open-weights model downloaded from Hugging Face at startup.

## What it does

`cvx-llamaserve` deploys the llama.cpp `llama-server`:

- One Deployment running the `ggml-org/llamacpp:server` container image.
- The model is downloaded from Hugging Face when the pod starts. `values.yaml`
  picks the model via `model:` — by default `ggml-org/Qwen3.5-0.8B-GGUF` — and
  tunes the server with `ctxSize:` (context window, default 4096 tokens) and
  `threads:`.
- One LoadBalancer Service exposing port `8080`, labelled with
  `networking.converix.io/vlan: pax` so Converix knows how to publish it.
- A NetworkPolicy that adapts to the Service type: with `LoadBalancer` (the
  default) it allows traffic from anywhere — every namespace in the cluster and
  clients outside it via `0.0.0.0/0`; with `ClusterIP` it allows only in-cluster
  traffic from every namespace.
- Startup probe (up to 5 minutes: 30 × 10s on `/health`) so the pod isn't killed
  while the model downloads and loads; liveness and readiness probes on
  `/health` after that.
- Resources are set by default (request 1 CPU / 3Gi, limit 2 CPU / 4Gi) — unlike
  the smaller apps in this repo, this one genuinely needs them.

Once the pod is ready, you get an OpenAI-compatible chat API plus llama.cpp's
built-in web UI at the service address. Swap `model:` in `values.yaml` for any
GGUF model on Hugging Face — smaller models load in seconds; bigger ones need
more memory and a longer startup allowance.

## What's in here

```text
cvx-llamaserve/
├── Makefile              # builds the offline bundle (image .tar + chart OCI archive)
├── cvx-llamaserve/       # the Helm chart
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

The fastest way to see `cvx-llamaserve` running is to build the offline bundle
and hand it to your Converix environment:

```sh
cd cvx-llamaserve
make
```

That produces exactly two deliverables under `dist/`:

- a Docker image archive (`.tar`) in **hybrid format** — an OCI layout
  (`index.json`, `oci-layout`, `blobs/`) plus Docker's legacy `manifest.json` in
  the same tar. Requires the containerd image store (Docker Desktop: Settings →
  General → "Use containerd for pulling and storing images"); the Makefile fails
  if the archive is missing either half.
- the chart as an OCI artifact (`cvx-llamaserve-<version>.tgz`) — a gzipped
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

1. Copy the `cvx-llamaserve/` directory to a new name, e.g. `cvx-myapp/`.
2. In `Chart.yaml`, change `name` (and `appVersion` if you want to track it).
3. In `values.yaml`, change `image.repository` and `image.tag` to your image.
4. Pick a `model:` that fits your memory budget, and adjust `resources` if you
   go bigger than the default.
5. Pick the right Converix `vlan` for your Service (`crew`, `seatbox`, or `pax`
   — these map to different audiences on the platform).
6. Run `make` to repackage and ship.

## Learn more

- [llama.cpp](https://github.com/ggml-org/llama.cpp) — the server and its API
- [GGUF models on Hugging Face](https://huggingface.co/models?library=gguf) —
  for choosing a `model:`
- [Kubernetes probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)
  — for how the startup probe buys time for model loading
