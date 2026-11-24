# =============================================================================
# Makefile for packaging a Helm chart together with its container image.
#
# Running `make` produces exactly two deliverables under dist/:
#   - a Docker image archive (.tar) in hybrid format (index.json +
#     manifest.json), pulled from the chart's configured image
#   - the chart as an OCI artifact (.tgz): a gzipped oci-archive of the
#     packaged Helm chart
# so both can be shipped/loaded into an air-gapped or offline environment.
#
# All intermediates (the packaged chart .tgz, the OCI layout dir, the image
# archive while it is being validated, and the unpack targets' inspection
# copies) live in build/, not dist/. Archives only move into dist/ once they
# are complete and have passed their format checks.
# =============================================================================

# Output directory for the two deliverables.
DIST_DIR := dist
# Build directory for intermediates and inspection copies.
BUILD_DIR := build

# Path to the Helm chart directory, auto-detected as the first subdirectory
# containing a Chart.yaml. Override on the command line with `make CHART_DIR=...`.
CHART_DIR ?= $(patsubst %/Chart.yaml,%,$(firstword $(wildcard */Chart.yaml)))

# Target platform for the pulled/saved image. Override for other architectures,
# e.g. `make PLATFORM=linux/arm64`.
PLATFORM ?= linux/amd64

# Image to bundle is taken from the chart's values.yaml (image.repository/image.tag).
IMAGE_NAME := $(shell yq '.image.repository' $(CHART_DIR)/values.yaml)
IMAGE_TAG  := $(shell yq '.image.tag' $(CHART_DIR)/values.yaml)
# Destination path for the saved image archive (.tar deliverable). Slashes
# in the repository name are flattened so the archive stays directly in dist/.
DOCKER_ARCHIVE := $(DIST_DIR)/$(subst /,-,$(IMAGE_NAME))-$(IMAGE_TAG).tar

# Packaged chart name/version come from Chart.yaml, so the .tgz path matches
# what `helm package` produces.
CHART_NAME    := $(shell yq '.name' $(CHART_DIR)/Chart.yaml)
CHART_VERSION := $(shell yq '.version' $(CHART_DIR)/Chart.yaml)
CHART_TGZ         := $(BUILD_DIR)/$(CHART_NAME)-$(CHART_VERSION).tgz
CHART_OCI_LAYOUT  := $(BUILD_DIR)/$(CHART_NAME)-$(CHART_VERSION)
CHART_OCI_ARCHIVE := $(DIST_DIR)/$(CHART_NAME)-$(CHART_VERSION).tgz
# The OCI config blob for the chart. Helm's config media type is +json, and
# Helm clients parse the config as JSON, so Chart.yaml must be converted
# (helm push serializes chart.Metadata to JSON the same way).
CHART_OCI_CONFIG  := $(BUILD_DIR)/$(CHART_NAME)-$(CHART_VERSION).config.json

# Every file in the chart directory, so the packaged .tgz rebuilds whenever
# anything in the chart changes (templates/ subdirectories, crds/, charts/
# subcharts, extra values files, ...).
CHART_FILES := $(shell find $(CHART_DIR) -type f)

# Directories the unpack targets extract archives into for inspection:
# build/unpacked/image/ (the Docker hybrid layout: index.json + manifest.json +
# blobs/) and build/unpacked/chart/ (the chart-as-OCI-artifact layout).
UNPACK_DIR       := $(BUILD_DIR)/unpacked
UNPACK_IMAGE_DIR := $(UNPACK_DIR)/image
UNPACK_CHART_DIR := $(UNPACK_DIR)/chart

.PHONY: all image chart chart-oci chart-oci-archive guard-image unpack-image unpack-chart inspect clean

# Default target: build both deliverables — the docker image archive and the
# chart as an OCI artifact archive.
all: image chart-oci-archive

# Create the output directories on demand (order-only prerequisites below).
$(DIST_DIR) $(BUILD_DIR):
	mkdir -p $@

# Fail-fast guard: the chart must name an image (repository and tag) to bundle.
guard-image:
	@if [ -z "$(IMAGE_NAME)" ]; then \
		echo "Error: image.repository is empty in $(CHART_DIR)/values.yaml" >&2; \
		exit 1; \
	fi
	@if [ -z "$(IMAGE_TAG)" ]; then \
		echo "Error: image.tag is empty in $(CHART_DIR)/values.yaml" >&2; \
		exit 1; \
	fi

# Pull the image with plain docker and save it as an uncompressed Docker-format
# tar (docker load reads .tar directly).
#
# The archive must be in Docker's *hybrid* format: an OCI layout (index.json,
# oci-layout, blobs/) plus a legacy manifest.json for backwards compatibility.
# That requires the containerd image store (Docker Desktop: Settings →
# General → "Use containerd for pulling and storing images"); with the classic
# store, `docker save` emits manifest.json only and the check below fails.
#
# The archive is written to build/ first and only moved into dist/ after the
# format checks pass, so a failed `docker save` can never leave a truncated
# archive behind that make would treat as up to date.
image: $(DOCKER_ARCHIVE)

# Rebuilds when the chart's image config changes; the guard runs first via
# order-only prerequisites.
$(DOCKER_ARCHIVE): $(CHART_DIR)/values.yaml | guard-image $(DIST_DIR) $(BUILD_DIR)
	docker pull --platform $(PLATFORM) $(IMAGE_NAME):$(IMAGE_TAG)
	docker save --platform $(PLATFORM) $(IMAGE_NAME):$(IMAGE_TAG) > $(BUILD_DIR)/image.tar
	@for member in index.json manifest.json; do \
		tar -tf $(BUILD_DIR)/image.tar | grep -qx $$member || { \
			echo "Error: image archive has no $$member — not in Docker hybrid format; enable the containerd image store" >&2; \
			exit 1; }; \
	done
	@mv $(BUILD_DIR)/image.tar $@

# Package the Helm chart into a versioned .tgz under build/.
# Sanity-check the chart (lint + render) before packaging.
chart: $(CHART_TGZ)

$(CHART_TGZ): $(CHART_FILES) | $(BUILD_DIR)
	helm lint $(CHART_DIR) --strict
	helm template $(CHART_DIR) > /dev/null
	helm package $(CHART_DIR) --destination $(BUILD_DIR)

# Package the chart as an OCI artifact in a local OCI layout directory.
# Pushes the packaged .tgz with Helm's media types (config = Chart.yaml,
# layer = chart content).
# The layout dir is removed first: re-pushing into an existing one appends
# another manifest to index.json instead of replacing.
chart-oci: $(CHART_OCI_LAYOUT)

$(CHART_OCI_LAYOUT): $(CHART_TGZ) $(CHART_DIR)/Chart.yaml | $(BUILD_DIR)
	rm -rf $@
	yq -o=json '.' -I=0 $(CHART_DIR)/Chart.yaml > $(BUILD_DIR)/chart.config.json
	oras push \
		--oci-layout \
		--image-spec v1.0 \
		--config $(BUILD_DIR)/chart.config.json:application/vnd.cncf.helm.config.v1+json \
		$@:$(CHART_VERSION) \
		$(CHART_TGZ):application/vnd.cncf.helm.chart.content.v1.tar+gzip

# Pack the OCI layout itself into a single gzipped oci-archive tar — the
# chart deliverable in dist/ (.tgz = gzipped tar). An oci-archive is just a tar of the layout
# (oci-layout, index.json, blobs/), so tar does the whole job. Entries must be
# canonical (no "./" prefix, no macOS "._*" AppleDouble files — hence
# COPYFILE_DISABLE=1): consumers match tar entry names exactly, so
# "./oci-layout" and "._oci-layout" break OCI-artifact detection.
chart-oci-archive: $(CHART_OCI_ARCHIVE)

$(CHART_OCI_ARCHIVE): $(CHART_OCI_LAYOUT) | $(DIST_DIR)
	COPYFILE_DISABLE=1 tar -czf $@ -C $(CHART_OCI_LAYOUT) oci-layout index.json blobs

# Unpack the Docker image archive into build/unpacked/image/ for inspection.
# Because the archive is in hybrid format, this yields an OCI layout
# (index.json, oci-layout, blobs/sha256/) plus Docker's legacy manifest.json
# alongside it. Inspect with:
#   skopeo inspect --raw oci:build/unpacked/image:$(IMAGE_TAG) | jq
#   cat build/unpacked/image/manifest.json | jq
unpack-image: $(DOCKER_ARCHIVE)
	rm -rf $(UNPACK_IMAGE_DIR)
	mkdir -p $(UNPACK_IMAGE_DIR)
	tar -xzf $(DOCKER_ARCHIVE) -C $(UNPACK_IMAGE_DIR)

# Unpack the chart archive into build/unpacked/chart/ for
# inspection. Yields a plain OCI layout (oci-layout, index.json, blobs/sha256/)
# Inspect with:
#   skopeo inspect --raw oci:build/unpacked/chart | jq
unpack-chart: $(CHART_OCI_ARCHIVE)
	rm -rf $(UNPACK_CHART_DIR)
	mkdir -p $(UNPACK_CHART_DIR)
	tar -xzf $(CHART_OCI_ARCHIVE) -C $(UNPACK_CHART_DIR)

# Unpack everything in dist/ into build/unpacked/ for inspection.
# Usage: `make inspect` (builds both deliverables first if missing).
inspect: unpack-image unpack-chart
	@echo "Unpacked for inspection:"
	@echo "  $(UNPACK_IMAGE_DIR)  — Docker hybrid layout (index.json + manifest.json + blobs/)"
	@echo "  $(UNPACK_CHART_DIR)  — chart OCI layout (oci-layout + index.json + blobs/)"

# Remove all build artifacts and intermediates.
clean:
	rm -rf $(DIST_DIR) $(BUILD_DIR)
