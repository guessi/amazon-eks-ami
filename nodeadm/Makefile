# Get the currently used golang install path (in GOPATH/bin, unless GOBIN is set)
ifeq (,$(shell go env GOBIN))
GOBIN=$(shell go env GOPATH)/bin
else
GOBIN=$(shell go env GOBIN)
endif

# Setting SHELL to bash allows bash commands to be executed by recipes.
# Options are set to exit when a recipe line exits non-zero or a piped command fails.
SHELL = /usr/bin/env bash -o pipefail
.SHELLFLAGS = -ec

.PHONY: all
all: crds generate fmt vet build

##@ General

# The help target prints out all targets with their descriptions organized
# beneath their categories. The categories are represented by '##@' and the
# target descriptions by '##'. The awk commands is responsible for reading the
# entire set of makefiles included in this invocation, looking for lines of the
# file as xyz: ## something, and then pretty-format the target and help. Then,
# if there's a line with ##@ something, that gets pretty-printed as a category.
# More info on the usage of ANSI control characters for terminal formatting:
# https://en.wikipedia.org/wiki/ANSI_escape_code#SGR_parameters
# More info on the awk command:
# http://linuxcommand.org/lc3_adv_awk.php

.PHONY: help
help: ## Display this help.
	@awk 'BEGIN {FS = ":.*##"; printf "\nUsage:\n  make \033[36m<target>\033[0m\n"} /^[a-zA-Z_0-9-]+:.*?##/ { printf "  \033[36m%-15s\033[0m %s\n", $$1, $$2 } /^##@/ { printf "\n\033[1m%s\033[0m\n", substr($$0, 5) } ' $(MAKEFILE_LIST)

##@ Development

.PHONY: clean
clean:
	rm -rf $(LOCALBIN)

.PHONY: crds
crds: controller-gen ## Generate CustomResourceDefinition objects.
	$(CONTROLLER_GEN) crd paths="./..." output:crd:artifacts:config=crds/

.PHONY: generate
generate: generate-code generate-doc ## Generate code and documentation.

.PHONY: generate-code
generate-code: controller-gen conversion-gen ## Generate code containing DeepCopy, DeepCopyInto, and DeepCopyObject method implementations.
	$(CONTROLLER_GEN) object paths="./..."
	$(CONVERSION_GEN) --input-dirs="./internal/api/bridge" --output-file-base=zz_generated.conversion --output-base="./" --go-header-file=/dev/null -v0

.PHONY: generate-doc
generate-doc: crd-ref-docs
	$(CRD_REF_DOCS) --config=doc/config.yaml --source-path=api/ --renderer=markdown --templates-dir=doc/templates --output-path=doc/api.md
	cat -s doc/api.md > doc/api.md.tmp
	sed '$$ d' doc/api.md.tmp > doc/api.md
	rm doc/api.md.tmp

.PHONY: fmt
fmt: goimports ## Run go fmt and goimports against code.
	go fmt ./...
	@$(GOIMPORTS) -w $(shell find . -type f -name '*.go' -not -path "./vendor/*")

.PHONY: vet
vet: ## Run go vet against code.
	go vet ./...

.PHONY: test
test: crds generate fmt vet ## Run go test against code.
	go test ./...

.PHONY: test-e2e
# the test infra container needs a linux executable
test-e2e: GOOS=linux
test-e2e: build ## Run e2e tests.
	test/e2e/run.sh

##@ Build

.PHONY: build
build: ## Build binary.
	GOOS=$(GOOS) go build -o $(LOCALBIN)/ ./cmd/...

.PHONY: run
run: build ## Run binary.
	go run cmd/nodeadm/main.go $(args)

.PHONY: wasm
wasm: ## Build WASM binary.
	GOOS=js GOARCH=wasm go build -o $(LOCALBIN)/nodeadm.wasm cmd/nodeadm-wasm/main.go

##@ Build Dependencies

## Location to install dependencies to
LOCALBIN ?= $(shell pwd)/_bin
$(LOCALBIN):
	mkdir -p $(LOCALBIN)

## Tool Binaries
CONTROLLER_GEN ?= $(LOCALBIN)/controller-gen
CONVERSION_GEN ?= $(LOCALBIN)/conversion-gen
CRD_REF_DOCS ?= $(LOCALBIN)/crd-ref-docs
GOIMPORTS ?= $(LOCALBIN)/goimports

## Tool Versions
CONTROLLER_TOOLS_VERSION ?= v0.14.0
CODE_GENERATOR_VERSION ?= v0.28.1
CRD_REF_DOCS_VERSION ?= v0.1.0
GOIMPORTS_VERSION ?= v0.34.0

tools: controller-gen conversion-gen crd-ref-docs goimports ## Install the toolchain.

.PHONY: controller-gen
controller-gen: $(CONTROLLER_GEN) ## Download controller-gen locally if necessary. If wrong version is installed, it will be overwritten.
$(CONTROLLER_GEN): $(LOCALBIN)
	test -s $(LOCALBIN)/controller-gen && $(LOCALBIN)/controller-gen --version | grep -q $(CONTROLLER_TOOLS_VERSION) || \
	GOBIN=$(LOCALBIN) go install sigs.k8s.io/controller-tools/cmd/controller-gen@$(CONTROLLER_TOOLS_VERSION)

.PHONY: conversion-gen
conversion-gen: $(CONVERSION_GEN) ## Download conversion-gen locally if necessary.
$(CONVERSION_GEN): $(LOCALBIN)
	test -s $(LOCALBIN)/conversion-gen || \
	GOBIN=$(LOCALBIN) go install k8s.io/code-generator/cmd/conversion-gen@$(CODE_GENERATOR_VERSION)

.PHONY: crd-ref-docs
crd-ref-docs: $(CRD_REF_DOCS) ## Download crd-ref-docs locally if necessary.
$(CRD_REF_DOCS): $(LOCALBIN)
	test -s $(LOCALBIN)/crd-ref-docs || \
	GOBIN=$(LOCALBIN) go install github.com/elastic/crd-ref-docs@$(CRD_REF_DOCS_VERSION)

.PHONY: goimports
goimports: $(GOIMPORTS) ## Download goimports locally if necessary.
$(GOIMPORTS): $(LOCALBIN)
	test -s $(LOCALBIN)/goimports || \
	GOBIN=$(LOCALBIN) go install golang.org/x/tools/cmd/goimports@$(GOIMPORTS_VERSION)

.PHONY: update-deps
update-deps:
	go get $(shell go list -f '{{if not (or .Main .Indirect)}}{{.Path}}{{end}}' -mod=mod -m all) && go mod tidy && go mod vendor
