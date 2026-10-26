# ============================================================
# Variables
# ============================================================
MAIN_PACKAGE   := ./cmd/server
BINARY_NAME    := server
BIN_DIR        := bin
BIN_PATH       := $(BIN_DIR)/$(BINARY_NAME)

VERSION        ?= $(shell git describe --tags --always --dirty 2>/dev/null || echo v0.0.0)
COMMIT         := $(shell git rev-parse --short HEAD 2>/dev/null || echo none)
BUILD_DATE     := $(shell date -u +'%Y-%m-%dT%H:%M:%SZ')

LDFLAGS        := -ldflags "-s -w \
  -X main.Version=$(VERSION) \
  -X main.Commit=$(COMMIT) \
  -X main.BuildDate=$(BUILD_DATE)"

# Bash with strict error handling (from Makefile-two)
SHELL          := bash
.SHELLFLAGS    := -eu -o pipefail -c

# Tool overrides (from Makefile-two)
GO             ?= go
GOLANGCI_LINT  ?= golangci-lint
DOCKER         ?= docker
COMPOSE_FILE   ?= compose.yaml

# Test configuration
TEST_TIMEOUT   ?= 5m
TEST_PKGS      ?= ./...

.DEFAULT_GOAL  := help

# ============================================================
# Build
# ============================================================
## build: Compile the binary
.PHONY: build
build:
	@mkdir -p $(BIN_DIR)
	CGO_ENABLED=0 $(GO) build -trimpath $(LDFLAGS) -o $(BIN_PATH) $(MAIN_PACKAGE)

## install: Install the binary to $GOPATH/bin
.PHONY: install
install:
	CGO_ENABLED=0 $(GO) install $(LDFLAGS) $(MAIN_PACKAGE)

## run: Build and run locally
.PHONY: run
run: build
	@./$(BIN_PATH)

## dev: Run with live reload (requires github.com/cosmtrek/air)
.PHONY: dev
dev:
	@air --build.cmd "make build" --build.bin "./$(BIN_PATH)"

# ============================================================
# Quality
# ============================================================
## test: Run all tests
.PHONY: test
test:
	$(GO) test -timeout $(TEST_TIMEOUT) -buildvcs -count=1 $(TEST_PKGS)

## test-race: Run tests with race detector
.PHONY: test-race
test-race:
	$(GO) test -race -timeout $(TEST_TIMEOUT) -buildvcs -count=1 $(TEST_PKGS)

## test/cover: Run tests and open coverage report
.PHONY: test/cover
test/cover:
	$(GO) test -race -coverprofile=/tmp/coverage.out $(TEST_PKGS)
	$(GO) tool cover -html=/tmp/coverage.out

## lint: Run golangci-lint
.PHONY: lint
lint:
	$(GOLANGCI_LINT) run --timeout=5m $(TEST_PKGS)

## fmt: Format all Go files
.PHONY: fmt
fmt:
	gofmt -s -w .
	goimports -w .

## fmt-check: Fail if Go files need formatting (for CI)
.PHONY: fmt-check
fmt-check:
	@files="$$(gofmt -s -l .)"; \
	if [ -n "$$files" ]; then \
		echo "ERROR: The following files need gofmt:"; \
		echo "$$files"; \
		exit 1; \
	fi
	@echo "Formatting check passed"

## vet: Run go vet
.PHONY: vet
vet:
	$(GO) vet $(TEST_PKGS)

## check-tools: Verify required development tools
.PHONY: check-tools
check-tools:
	@command -v $(GO) >/dev/null || { echo "ERROR: Go is not installed"; exit 1; }
	@echo "Go: $$($(GO) version)"
	@if command -v $(GOLANGCI_LINT) >/dev/null 2>&1; then \
		echo "$(GOLANGCI_LINT): installed"; \
	else \
		echo "WARNING: $(GOLANGCI_LINT) is not installed"; \
	fi
	@if command -v $(DOCKER) >/dev/null 2>&1; then \
		echo "Docker: installed"; \
	else \
		echo "WARNING: Docker is not installed"; \
	fi

## audit: Full quality gate (used in CI)
.PHONY: audit
audit: fmt-check vet lint test-race
	$(GO) mod tidy -diff
	$(GO) mod verify
	$(GO) run golang.org/x/vuln/cmd/govulncheck@latest $(TEST_PKGS)

# ============================================================
# Docker
# ============================================================
## docker/build: Build the container image
.PHONY: docker/build
docker/build:
	$(DOCKER) build --build-arg VERSION=$(VERSION) --build-arg COMMIT=$(COMMIT) \
		-t $(BINARY_NAME):$(VERSION) .

## docker/release: Tag and push image
.PHONY: docker/release
docker/release:
	$(DOCKER) build --build-arg VERSION=$(VERSION) --build-arg COMMIT=$(COMMIT) \
		-t $(BINARY_NAME):$(VERSION) -t $(BINARY_NAME):latest .
	$(DOCKER) push $(BINARY_NAME):$(VERSION)
	$(DOCKER) push $(BINARY_NAME):latest

## docker/up: Start services using Docker Compose
.PHONY: docker/up
docker/up:
	$(DOCKER) compose -f $(COMPOSE_FILE) up -d --build

## docker/down: Stop Docker Compose services
.PHONY: docker/down
docker/down:
	$(DOCKER) compose -f $(COMPOSE_FILE) down

## docker/logs: Follow Docker Compose logs
.PHONY: docker/logs
docker/logs:
	$(DOCKER) compose -f $(COMPOSE_FILE) logs -f

# ============================================================
# Release (cross-compile)
# ============================================================
## release: Build binaries for linux/darwin/windows
.PHONY: release
release:
	@mkdir -p dist
	CGO_ENABLED=0 GOOS=linux   GOARCH=amd64 $(GO) build -trimpath $(LDFLAGS) -o dist/$(BINARY_NAME)-linux-amd64 $(MAIN_PACKAGE)
	CGO_ENABLED=0 GOOS=linux   GOARCH=arm64 $(GO) build -trimpath $(LDFLAGS) -o dist/$(BINARY_NAME)-linux-arm64 $(MAIN_PACKAGE)
	CGO_ENABLED=0 GOOS=darwin  GOARCH=amd64 $(GO) build -trimpath $(LDFLAGS) -o dist/$(BINARY_NAME)-darwin-amd64 $(MAIN_PACKAGE)
	CGO_ENABLED=0 GOOS=darwin  GOARCH=arm64 $(GO) build -trimpath $(LDFLAGS) -o dist/$(BINARY_NAME)-darwin-arm64 $(MAIN_PACKAGE)
	CGO_ENABLED=0 GOOS=windows GOARCH=amd64 $(GO) build -trimpath $(LDFLAGS) -o dist/$(BINARY_NAME)-windows-amd64.exe $(MAIN_PACKAGE)

# ============================================================
# Housekeeping
# ============================================================
## tidy: Tidy modules and format
.PHONY: tidy
tidy:
	$(GO) mod tidy -v
	$(GO) fmt $(TEST_PKGS)

## clean: Remove build artifacts
.PHONY: clean
clean:
	rm -rf $(BIN_DIR) dist /tmp/coverage.out

## help: Show this help
.PHONY: help
help:
	@echo "Usage: make <target>"
	@echo ""
	@grep -E '^## ' $(MAKEFILE_LIST) \
		| awk 'BEGIN {FS="## "}; {printf "  \033[36m%-14s\033[0m %s\n", $$1, $$2}'