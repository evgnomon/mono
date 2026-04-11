# Monorepo Structure

A self-similar, recursive directory schema for multi-language, multi-process repositories.
Inspired by the Linux kernel's architecture, adapted for application-level software.

## Schema

Every level of the tree follows the same shape. Directories appear only when earned.

```
<name>/
├── Makefile          # build orchestration, recurses into children
├── README.md         # what is this — code and doc are the same
├── lib/              # code, layered — build order defines dependency direction
├── cmd/              # entry points — if present, this is runnable
├── api/              # contracts: protobuf, openapi, jsonschema
├── arch/             # platform-specific implementations of the same interface
├── deploy/           # dockerfiles, k8s, terraform, ansible
│   ├── configs/      # config templates and env-specific defaults
│   └── init/         # systemd, supervisord, sysv units
├── scripts/          # build-time helper scripts
├── third_party/      # vendored external code as git submodules, built and linked locally
├── test/             # integration, e2e — unit tests live with the code
│   └── testdata/     # fixtures and test data
```

All levels output to a single prefix at the root, structured like a Linux root
filesystem. The build tree is **separated from source** and lives only at the
top level:

```
build/                    # root-level only — all levels output here
├── bin/                  # executables and scripts
├── lib/                  # shared libraries, plugins, JARs
├── include/              # generated headers, proto stubs
├── share/                # arch-independent: assets, APKs, web bundles
└── etc/                  # config templates

dist/                     # root-level only — not checked in, temporary
                          # packaging artifacts: .deb, .rpm, .tar.gz, .zip
```

Package `build/` into `dist/` as a `.deb`, `.rpm`, or `.tar.gz`. Cache the
build tree by content hash for incremental builds. Layer it into a container
with `COPY build/ /`.

Cross-compiled binaries are prefixed with their target name (e.g.
`arm-sensor-gateway`) and kept in `build/bin/` alongside native binaries
when they need to be distributed together.

## Principles

### Self-similarity

The structure is recursive. The workspace root has the same directories as a domain
inside it, which has the same directories as a leaf library. You never have to learn
a new layout — it's the same pattern at every depth.

This is not about building the entire system from source. You extend a base
distribution — a Debian system with packages already installed. To build a
component, `cd` into its directory and run `make`. The build picks up what it
needs from the system and from `$(PREFIX)`. A single stamp file can guard any
expensive action so it runs only when inputs actually change.

### Merge without conflict

Each domain is a top-level directory named after itself. When you bring in another
repository, it becomes a sibling directory. Nothing overwrites anything because the
name **is** the namespace. The root contains only generic glue (`Makefile`, `README.md`)
that the imported repo wouldn't have at its own root.

### One Makefile, no extras

Each level has a single `Makefile`. Variables flow down via `export` — no `vars.mk`,
no `-include` chains, no relative path fragility. The top-level Makefile exports
shared variables (`REGISTRY`, `VERSION`), and every child `$(MAKE)` inherits them
automatically.

```makefile
# Top-level Makefile
PREFIX   ?= $(CURDIR)/build
REGISTRY ?= ghcr.io/acme
VERSION  ?= $(shell git rev-parse --short HEAD)

export PREFIX REGISTRY VERSION

LIBS := $(wildcard lib/*/)

# Each lib is a target — Make's -j flag parallelizes them
$(LIBS):
	$(MAKE) -C $@ $(TARGET)

# Declare inter-lib dependencies
lib/api/:    lib/core/
lib/auth/:   lib/core/
lib/gateway/: lib/auth/ lib/api/

build test lint:
	$(MAKE) TARGET=$@ $(LIBS)

clean:
	rm -rf $(PREFIX)
	@for d in $(LIBS); do $(MAKE) -C $$d clean; done

.PHONY: build test lint clean $(LIBS)
```

Run `make -j$(nproc) build`. Make sees the DAG — `lib/core/` builds first,
`lib/auth/` and `lib/api/` build in parallel, `lib/gateway/` waits for both.
No shell loop, no serial bottleneck.

At a leaf with a binary:

```makefile
BIN := $(notdir $(CURDIR))
SRC := $(shell find . -name '*.go' -not -path './test/*')

build:
	@mkdir -p $(PREFIX)/bin
	go build -o $(PREFIX)/bin/$(BIN) ./cmd/...

.stamps:
	@mkdir -p .stamps

.stamps/test: .stamps $(SRC) $(shell find . -name '*_test.go')
	go test ./... -race
	@touch $@

test: .stamps/test

clean:
	rm -f $(PREFIX)/bin/$(BIN)
	rm -rf .stamps
```

Make sees the stamp's mtime against the source files. If nothing changed, the
test target is a no-op. After a change, only the affected component re-runs.

### Separated build tree

All levels output to a single `$(PREFIX)` at the root. No per-component `build/`
directories, no symlinks. Source and output are fully separated.

- `$(PREFIX)/bin/` — compiled binaries, installed scripts
- `$(PREFIX)/lib/` — shared libraries, Go plugins, JARs
- `$(PREFIX)/include/` — generated headers, protobuf stubs
- `$(PREFIX)/share/` — arch-independent data: web bundles, APKs, assets
- `$(PREFIX)/etc/` — config templates

The top-level Makefile sets `PREFIX ?= $(CURDIR)/build` and exports it. Every
child Makefile inherits it and writes directly to the appropriate subdirectory.
The result is a self-contained tree you can:

- **Package** as a `.deb`, `.rpm`, or `.tar.gz`
- **Cache** by content hash for incremental and distributed builds
- **Layer** into a container image with `COPY build/ /`

### Cached builds

You do not rebuild the world for every checkout. A nightly CI job publishes the
full `build/` tree to an SFTP server (or any artifact store). A fresh checkout
downloads the cached build and starts from there — only changed components
rebuild. This means a PR build touches only what the PR actually changed, not
the entire million-file tree.

### Dependencies target the build tree

Components never import each other's source. They depend on what is installed
in `$(PREFIX)`:

- A service that calls another service depends on the binary in `$(PREFIX)/bin/`
- A Python model that loads a shared library depends on `$(PREFIX)/lib/`
- Generated protobuf stubs install to `$(PREFIX)/include/`
- OpenAPI specs and JSON schemas install to `$(PREFIX)/share/`

The `api/` source directory contains definitions (`.proto`, `.yaml`, `.json`).
The build step compiles them and installs outputs into the build tree. Consumers
depend on the build tree, not on `api/` source. This eliminates cross-language
dependency problems — every language depends on the same build artifacts, not
on each other.

### Presence signals purpose

A directory's presence tells you what a component is:

| Has          | Meaning                                    |
|--------------|--------------------------------------------|
| `cmd/`       | Runnable — has a binary entry point        |
| `deploy/`    | Deployable — runs as a service             |
| `arch/`      | Multi-platform — swappable implementations |
| `third_party/` | Interfaces with external systems         |
| Neither      | Pure library — imported by others          |

### Tests live at two levels

**Unit tests** sit next to the code they test: `auth.go` → `auth_test.go`, with
`testdata/` alongside for fixtures. **Integration and e2e tests** that cross
library boundaries live in the top-level `test/` directory.

Expensive tests (gating tests, full e2e suites) are separate Make targets —
not part of the default `make test`. They run when explicitly requested, such
as in CI before a merge. Running all dependents when something changes is
correct — the point is to know what broke.

Each level uses stamp files for invalidation. Top-level integration tests:

```makefile
SRC := $(shell find ../lib/ -name '*.go')

.stamps:
	@mkdir -p .stamps

.stamps/integration: .stamps $(SRC) $(shell find integration/ -name '*.go')
	go test ./integration/...
	@touch $@

.stamps/e2e: .stamps $(SRC) $(shell find e2e/ -name '*.go')
	go test ./e2e/...
	@touch $@

test: .stamps/integration .stamps/e2e

clean:
	rm -rf .stamps
```

A change to `lib/auth/` re-stamps `lib/auth/.stamps/test` and
`test/.stamps/integration`, but leaves `lib/fraud/.stamps/test` untouched.
CI caches `.stamps/` directories between runs.

### One product, one owner

There are no team boundaries. The repository builds one thing. Visibility rules
and access control are unnecessary — if code is in the repo, it is part of the
product.

**Documentation is the contract.** Anything documented properly is a stable
interface. Anything undocumented can be changed or dropped without notice. This
replaces visibility rules: you don't need `BUILD` file annotations when the
docs *are* the annotation.

**Ownership follows contribution.** The owner of a component is whoever has the
most commits to it (`git shortlog -sn -- lib/auth/`). No `OWNERS` files to
maintain — the history is the source of truth.

**Shallow clones only.** Never fetch the entire history. `git clone --depth=1`
or sparse checkout for CI. Full history is a query against the remote, not
something every checkout carries. A single person can look at the large
picture and let others know what is going on.

## Relation to the Linux kernel

This structure mirrors the kernel's layout, consolidated:

| Ours           | Kernel                    | Role                              |
|----------------|---------------------------|-----------------------------------|
| `Makefile`     | `Makefile`                | Build orchestration               |
| `README.md`    | `README`                  | What is this                      |
| `build/`       | `build/`                  | Output tree (root-level, FHS-like)|
| `lib/`         | `lib/` `fs/` `net/` `mm/` | Code libraries                    |
| `cmd/`         | `init/main.c`             | Entry points                      |
| `api/`         | `include/`                | Contracts between components      |
| `arch/`        | `arch/`                   | Platform-specific implementations |
| `deploy/init/` | `init/`                   | Process initialization            |
| `scripts/`     | `scripts/`                | Build-time helpers                |
| `third_party/` | `drivers/`                | Interfacing with external things  |
| `test/`        | `tools/testing/`          | Cross-component tests             |

The kernel spreads its domains (`fs/`, `net/`, `mm/`, `crypto/`) at the top level
because it grew over 30 years. This schema groups them under `lib/` and applies the
same structure recursively.

In the kernel, `include/` defines headers — the contract between subsystems. That's
exactly what `api/` is: protobuf, openapi, or jsonschema definitions that specify
how components talk to each other.

The kernel's `drivers/` is code that interfaces with external hardware. `third_party/`
serves the same purpose: code that talks to things you don't own. It is added as git
submodules — pinned to exact commits for reproducible builds — built locally, and
its outputs install to `$(PREFIX)` like everything else.

The kernel's `arch/` provides multiple implementations of the same interface (x86 vs
arm64 `memcpy`, for example). `arch/` in this schema does the same: `arch/azure/` vs
`arch/aws/` for storage backends, `arch/cuda/` vs `arch/cpu/` for inference, selected
at build time.

## Example: multi-domain workspace

```
acme/
├── Makefile
├── README.md
├── build/                         # output tree — mirrors Linux FHS
│   ├── bin/                       # auth, api, jobs, gateway, ...
│   ├── lib/                       # libcore.so, libstorage.so, ...
│   ├── include/                   # generated proto stubs
│   ├── share/
│   │   ├── openapi/               # generated OpenAPI specs
│   │   ├── jsonschema/            # generated JSON schemas
│   │   ├── web-app/               # static SPA bundle
│   │   ├── admin/                 # admin dashboard bundle
│   │   └── android/               # android-app.apk
│   └── etc/                       # config templates
├── lib/
│   ├── core/                      # Go — shared types, errors, logging
│   ├── auth/                      # Go — middleware, JWT (has cmd/)
│   ├── api/                       # Go — HTTP/gRPC server (has cmd/, deploy/)
│   ├── jobs/                      # Go — background workers (has cmd/, deploy/)
│   ├── storage/                   # Go — generic interface
│   │   └── arch/                  #   azure/, aws/, local/ implementations
│   ├── gateway/                   # Go — edge service (has cmd/, deploy/)
│   │   └── third_party/           #   stripe/, twilio/ integrations
│   ├── ui-components/             # TypeScript — shared React components
│   ├── web-app/                   # TypeScript — main SPA (has cmd/, deploy/)
│   ├── admin/                     # TypeScript — internal dashboard (has cmd/, deploy/)
│   ├── sdk/                       # TypeScript — public client SDK
│   ├── mobile-core/               # Kotlin — shared types, networking
│   ├── android/                   # Kotlin — Android app (has deploy/)
│   ├── pipeline/                  # Scala — data pipelines (has cmd/, deploy/)
│   │   └── arch/                  #   spark/, flink/ runtimes
│   ├── recommender/               # Python — ML model (has cmd/, deploy/)
│   │   ├── arch/                  #   cuda/, cpu/, onnx/ backends
│   │   └── third_party/           #   vendor model weights
│   └── fraud/                     # Python — fraud detection (has cmd/, deploy/)
├── api/
│   ├── protobuf/
│   ├── openapi/
│   └── jsonschema/
├── arch/                          # top-level platform defaults
│   ├── linux/
│   ├── darwin/
│   └── windows/
├── deploy/
│   ├── ansible/
│   ├── docker/
│   ├── k8s/
│   ├── terraform/
│   ├── configs/
│   └── init/
├── scripts/
├── third_party/                   # repo-wide vendored code
├── test/
│   ├── integration/
│   ├── e2e/
│   └── testdata/
└── dist/                          # not checked in — packaging artifacts
```

## Adding another domain

Copy or `git subtree add` a repo that follows this schema as a new top-level directory:

```
acme/
├── ...
├── lib/
├── bravo/          ← new domain, same structure inside
│   ├── Makefile
│   ├── README.md
│   ├── lib/
│   ├── api/
│   ├── deploy/
│   └── ...
└── ...
```

Register it in the root Makefile or let wildcard discovery pick it up. Nothing existing
is touched.
