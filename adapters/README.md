adapters/
Thin translation layers between external tools, services, and APIs and our own
abstractions, names, and conventions.
Purpose
Each subdirectory here wraps a piece of external machinery — a CI system, a
cloud CLI, a configuration management tool, a device API — and re-exposes it
through our internal vocabulary. Callers elsewhere in the codebase should not
need to know the foreign tool's terminology, flags, or quirks; they interact
with our abstractions, and the adapter does the translation.
The name follows the ports-and-adapters (hexagonal architecture) sense:
our side defines the shape it wants; the adapter implements that shape against
whatever the outside world actually provides.
What belongs here

Wrappers around third-party CLIs (az, gh, ansible-playbook, skopeo, ...)
GitHub Actions / CI glue that drives external tools on our behalf
API clients that map foreign resource models onto our domain model
Shims that normalize behavior across platforms or tool versions

What does not belong here

Pure business logic or domain code — that lives alongside the abstractions
the adapters serve, not here
Standalone utilities with no external dependency to adapt
Generic helpers (logging, config parsing, retries) — those go in shared
libraries, not under an adapter

Layout
Adapters are grouped by category, where the category is the interface
they implement on our side. Each category directory holds one subdirectory
per external implementation:
adapters/
  <category>/                 # our interface (e.g. runner, provisioner)
    <external-system>/        # one adapter per external backend
      README.md
      ...
When an adapter bridges two external systems — for example, an Ansible
playbook that drives GitHub Actions — the directory name encodes both, in
<caller>-<backend> order (the caller is whichever side we invoke first;
the backend is what ends up doing the work through it):
adapters/runner/ansible-github-actions/
Read it as: "inside the runner category, the adapter where Ansible drives
GitHub Actions." The category tells you which interface is satisfied; the
directory name tells you what external parts are involved and in which
direction.
Example tree
adapters/
  runner/
    ansible-github-actions/        # Ansible driving GitHub Actions workflows
    ansible-gitlab-ci/             # Ansible driving GitLab CI pipelines
    ansible-shell/                 # Ansible driving plain shell steps
    local-ansible/                 # ansible-playbook invoked directly
  provisioner/
    ansible/                       # our Provisioner over raw Ansible
    terraform/                     # our Provisioner over Terraform
  device-registry/
    azure-iot/                     # Azure IoT Hub
    aws-iot/                       # AWS IoT Core
  image-store/
    skopeo/                        # skopeo against any registry
    docker-registry-v2/            # direct v2 HTTP API
  auth/
    keycloak/
    azure-entra/
  secret-store/
    ansible-vault/
    azure-key-vault/
    hashicorp-vault/
Multiple adapters in the same category are expected and encouraged — that is
the point of having the category. Callers depend on the interface
(Runner, Provisioner, ...) and are agnostic to which subdirectory
actually backs it at runtime.
Picking a category name

Use the singular noun form of the interface (runner, not runners;
image-store, not image-stores).
Use kebab-case for multi-word categories so directory names stay shell-
friendly across the codebase.
If you are about to add an adapter and no category fits, stop and define
the interface first. Adapters without a category signal that the domain
side has not decided what contract it wants.

Other conventions

The adapter's public surface uses our names for things. Foreign
terminology (their resource IDs, flag names, enum values) stays inside the
adapter and is translated at the boundary.
Keep adapters thin. If an adapter starts accumulating business rules, the
rules belong on the inside of the hexagon, not here.
Each adapter should be independently testable against a fake or recorded
fixture of the external system so the rest of the codebase can be tested
without touching the real thing.

Adding a new adapter

Decide which category (interface) the adapter belongs to. If none fits,
define the interface first — the interface lives with the domain code,
not under adapters/. Then create the category directory here.
Create adapters/<category>/<external-system>/ (or
adapters/<category>/<caller>-<backend>/ if the adapter bridges two
external systems) with a short README.md describing: which external
system(s) it wraps, which version(s) it targets, and the mapping between
their names and ours.
Write the adapter to satisfy the category's interface.
Add fixtures or a fake sufficient for tests elsewhere in the tree to run
without network or external binaries.
