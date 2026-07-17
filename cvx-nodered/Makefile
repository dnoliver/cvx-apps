# =============================================================================
# Makefile for packaging a Helm chart together with its container image.
#
# Running `make` produces, under dist/:
#   - a Docker image archive (.tar) pulled from the chart's configured image
#   - a packaged Helm chart (.tgz)
# so both can be shipped/loaded into an air-gapped or offline environment.
# =============================================================================

# Output directory for build artifacts (image tarball + packaged chart).
DIST_DIR := dist

# Path to the Helm chart directory, auto-detected as the first subdirectory
# containing a Chart.yaml. Override on the command line with `make CHART_DIR=...`.
CHART_DIR ?= $(patsubst %/Chart.yaml,%,$(firstword $(wildcard */Chart.yaml)))

# Target platform for the pulled/saved image. Override for other architectures,
# e.g. `make PLATFORM=linux/arm64`.
PLATFORM ?= linux/amd64

# Image to bundle is taken from the chart's values.yaml (image.repository/image.tag).
IMAGE_NAME := $(shell yq '.image.repository' $(CHART_DIR)/values.yaml)
IMAGE_TAG := $(shell yq '.image.tag' $(CHART_DIR)/values.yaml)
# Destination path for the saved image archive.
DOCKER_ARCHIVE := $(DIST_DIR)/$(IMAGE_NAME)-$(IMAGE_TAG).tar

.PHONY: all image chart clean

# Default target: build both the image archive and the packaged chart.
all: image chart

# Create the output directory on demand (order-only prerequisite below).
$(DIST_DIR):
	mkdir -p $(DIST_DIR)

# Pull the image for the target platform and save it as a tarball under dist/.
# Fails early if the chart does not specify an image tag.
image: $(DIST_DIR)
	@if [ -z "$(IMAGE_TAG)" ]; then \
		echo "Error: image.tag is empty in $(CHART_DIR)/values.yaml" >&2; \
		exit 1; \
	fi
	docker pull --platform $(PLATFORM) $(IMAGE_NAME):$(IMAGE_TAG)
	mkdir -p $(dir $(DOCKER_ARCHIVE))
	docker save --platform $(PLATFORM) -o $(DOCKER_ARCHIVE) $(IMAGE_NAME):$(IMAGE_TAG)

# Package the Helm chart into a versioned .tgz under dist/.
# Sanity-check the chart (lint + render) before packaging.
chart: $(DIST_DIR)
	helm lint $(CHART_DIR) --strict
	helm template $(CHART_DIR) > /dev/null
	helm package $(CHART_DIR) --destination $(DIST_DIR)

# Remove all build artifacts.
clean:
	rm -rf $(DIST_DIR)
