PKG = github.com/kyma-project/kyma/components/function-controller

# Image URL to use all building/pushing image targets
APP_NAME ?= function-controller
IMG ?= $(APP_NAME):latest
IMG-CI = $(DOCKER_PUSH_REPOSITORY)$(DOCKER_PUSH_DIRECTORY)/$(APP_NAME):$(DOCKER_TAG)

all: test manager

# Resolve dependencies
resolve:
	dep ensure -v -vendor-only

.PHONY: pull-licenses
pull-licenses:
ifdef LICENSE_PULLER_PATH
	bash $(LICENSE_PULLER_PATH)
else
	mkdir -p licenses
endif

# Run tests
# Default is 20s - available since controller-runtime 0.1.5
test: export KUBEBUILDER_CONTROLPLANE_START_TIMEOUT = 2m
# Default is 20s - available since controller-runtime 0.1.5
test: export KUBEBUILDER_CONTROLPLANE_STOP_TIMEOUT = 2m
test: generate fmt vet
	go test -race -v ./pkg/... ./cmd/... -coverprofile cover.out
	@echo "Total test coverage: $$(go tool cover -func=cover.out | grep total | awk '{print $$3}')"

# Generate code
generate: resolve
	go generate ./pkg/... ./cmd/...

# Run go fmt against code
fmt:
	go fmt ./pkg/... ./cmd/...

# Run go vet against code
vet:
	go vet ./pkg/... ./cmd/...

qt:
	go test -v ./pkg/... ./cmd/... -coverprofile cover.out
	@echo "Total test coverage: $$(go tool cover -func=cover.out | grep total | awk '{print $$3}')"

# Build manager binary
manager: generate fmt vet
	go build -o bin/manager $(PKG)/cmd/manager

# NOTE: does not work anymore until https://github.com/kubernetes-sigs/kubebuilder/issues/400 is implemented
# Run against the configured Kubernetes cluster in ~/.kube/config
run: generate fmt vet
	go run ./cmd/manager/main.go

# Install CRDs into a cluster
install:
	kubectl apply -f config/crds/

# Uninstall CRDs from cluster
uninstall:
	kubectl delete -f config/crds/

# Generate manifests e.g. CRD, RBAC etc.
manifests:
	go run vendor/sigs.k8s.io/controller-tools/cmd/controller-gen/main.go all

# Deploy controller in the configured Kubernetes cluster in ~/.kube/config

sedbackupflag = -i
ifeq ($(shell uname -s),Darwin)
	sedbackupflag +=
endif

deploy: manifests install bin/kustomize ko
	bin/kustomize build config/default | ko apply -f -

removewebhook:
	-kubectl delete mutatingwebhookconfigurations.admissionregistration.k8s.io function-webhook.serverless.kyma-project.io

undeploy: uninstall removewebhook bin/kustomize ko
	bin/kustomize build config/default | ko delete  -f -

# Build the docker image
.PHONY: docker-build
docker-build: resolve pull-licenses manifests test
	docker build . -t $(IMG)

# Push the docker image
.PHONY: docker-push
docker-push:
ifdef DOCKER_PUSH_REPOSITORY
	docker tag $(IMG) $(IMG-CI)
	docker push $(IMG-CI)
else
	docker push $(IMG)
endif

.PHONY: ci-pr
ci-pr: docker-build docker-push

.PHONY: ci-master
ci-master: docker-build docker-push

.PHONY: ci-release
ci-release: docker-build docker-push

.PHONY: path-to-referenced-charts
path-to-referenced-charts:
	@echo "resources/function-controller"

# used by 'go install'
GOPATH ?= $(shell go env GOPATH)
export GOBIN := $(GOPATH)/src/$(PKG)/bin

.PHONY: bin/kustomize
bin/kustomize:
	@go install ./vendor/sigs.k8s.io/kustomize

.PHONY: ko
ko:
	@go get -u github.com/google/ko/cmd/ko

