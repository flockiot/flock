# Flock — Platform Specification
### Seed Document · v0.4

---

## What Is Flock

Flock is an open-source, OS-agnostic device fleet management platform. Teams use it to deploy, update, monitor, and remotely access fleets of Linux devices running containerized workloads — without being locked to a proprietary OS, container runtime, or cloud provider.

The closest comparable product is Balena. Flock improves on every layer: a modern Go backend, a Rust on-device agent, a runtime-agnostic container model, a vendor-agnostic data layer, a professional-grade release and rollout system, customer-facing event streaming, and a fully open self-hosted experience that is first-class not an afterthought.

The platform is being built with LLM assistance, guided by a human architect. Decisions in this document are final unless explicitly revisited. Implementation details (schemas, libraries, internal structure) are intentionally deferred to per-component planning sessions.

---

## Guiding Principles

- Work in small, verifiable increments. Each task produces something runnable or testable.
- Prefer explicit and readable over clever.
- No stubs or scaffolding unless explicitly asked.
- Tests are part of every task, not a follow-up.
- When a decision is not in this document, ask before assuming.

---

## Repositories

**`flock-api`** — Go backend. One binary, multiple runtime targets.

**`flock-agent`** — Rust on-device agent (`flockd`). Single static binary, no runtime dependencies.

**`flock-ui`** — React SPA for the dashboard plus static MDX documentation, served by nginx. No SSR, no framework server process.

**`flock-cli`** — Rust CLI (`flock`). Single static binary. The operator and developer interface to the platform.

**`flock-base`** — Helm base chart. What someone uses to self-host Flock on any Kubernetes cluster.

**`flock-local`** — Helm values overrides for local development. Runs the full platform on Docker Desktop Kubernetes (or kind/k3d). Single-binary mode.

**`flock-os`** — OS image recipes built with debos (Debian Bookworm base). Contains recipes for official images, community-contributed recipes, overlay files (flockd config, cloud-init, systemd units), and build scripts. GitHub Actions builds images on tag and uploads to object storage.

**`flock-staging`** — Helm values overrides for staging. No application code.

**`flock-production`** — Helm values overrides for production. No application code.

### CI/CD

Everything runs through **GitHub Actions**.

- `flock-api`: test → lint → build Docker image → push to GHCR on tag
- `flock-agent`: test → cross-compile all target architectures → publish binaries to GitHub Releases on tag
- `flock-cli`: test → cross-compile all target architectures → publish binaries to GitHub Releases on tag
- `flock-ui`: test → build static assets → build nginx Docker image → push to GHCR on tag
- `flock-os`: build debos images for all official targets → upload to object storage on tag
- `flock-staging` / `flock-production`: on push to main, a workflow authenticates to the target cluster and runs `helm upgrade --install`

---

## flock-api — Backend

### Architecture Pattern

One Go binary. Multiple runtime targets selected at startup via `--target` flag or `FLOCK_TARGET` env var. Same model as Grafana Loki components.

`--target=all` in local development. In production each target is a separate Kubernetes Deployment that scales independently. One codebase, one CI pipeline, one Docker image.

### Targets

**`api`** — User-facing REST API. Authentication, all resource CRUD, SSE for log streaming, WebSocket for the web terminal. What the CLI and UI talk to.

**`ingester`** — Subscribes to device topics on the MQTT broker. Every heartbeat, telemetry report, log line, and event flows through here. Writes device state to the KV store, batches telemetry to ClickHouse, publishes structured events to internal MQTT topics. Scales horizontally via shared subscriptions — any ingester handles any device.

**`scheduler`** — Rollout execution engine. Subscribes to internal MQTT topics, evaluates active rollouts, advances or pauses them based on policy and health signals, publishes desired state to device topics on the broker. Single replica with leader election.

**`builder`** — Builds container images when a new release is created. Multi-architecture output (amd64, arm64, armv7). Scales to zero when idle.

**`delta`** — Generates binary delta patches between OCI image layer versions and stores them in object storage. Scales to zero when idle.

**`registry-proxy`** — A thin proxy in front of Harbor. Serves delta manifests that tell the agent which deltas are available for a given image update. Falls through to Harbor transparently for normal pulls.

**`tunnel`** — WireGuard overlay network. Every device gets a WireGuard peer and private IP on registration. Handles SSH proxying for `flock ssh <device>` and reverse tunnels for public device URLs. Horizontal scaling handled at implementation via shared peer state in the KV store.

**`proxy`** — Public device URL feature. Routes `<uuid>.devices.flock.io` HTTP/HTTPS to the correct device port through the tunnel layer. Wildcard DNS, dynamic routing, no static config per device.

**`events-gateway`** — Customer-facing event streaming. Subscribes to customer-facing MQTT topics and fans them out to webhook endpoints and WebSocket connections.

### Data Layer

Each concern has a different access pattern and lives in a different store. Specific schemas and query patterns are defined in per-component planning sessions.

**PostgreSQL** — Relational, slowly-changing, strongly consistent data. Organizations, users, memberships, fleets, devices (registration metadata only), releases, deployments, rollout state, provisioning keys, API keys, environment variables, event subscriptions, build jobs.

**Pluggable KV store** — Live device state only: reported state (what the device is running now, written by ingester on every heartbeat) and desired state (what the scheduler wants it to run). Interface-based in Go with implementations for Valkey, ScyllaDB, and DynamoDB — selected via config. The interface is defined from day one. Valkey is the only required implementation for v1; the others follow when there is a concrete scale requirement.

**ClickHouse** — All time-series data from devices: connectivity events, OTA progress, system telemetry (CPU, memory, disk, network, temperature), container logs. Append-only, high write throughput, batched writes from the ingester.

**Loki** — Audit log storage. Audit events are structured, append-only, and have different retention and query patterns from operational data. Loki handles this naturally — audit events are log streams labeled by org, actor, resource type, and action. Already in the stack. The audit log never touches PostgreSQL.

**Harbor** — OCI image registry. CNCF-graduated, production-proven. Provides RBAC, garbage collection, and vulnerability scanning via Trivy. Runs as a Helm subchart. The `registry-proxy` target sits in front for delta serving.

**S3-compatible object storage** — Delta patch files, build logs, OS image artifacts. MinIO locally, any S3-compatible store in production. Configured via endpoint URL, not vendor-specific.

**MQTT broker** — Central message bus for all async communication: device-to-cloud, cloud-to-device, and internal service-to-service. Self-hosted EMQX (or compatible broker) as a Helm subchart. All backend targets are MQTT clients — no target accepts device connections directly.

### MQTT Topic Conventions

A deliberate namespace boundary enforces the difference between device topics, internal topics, and customer-facing topics.

**`devices/{id}/+`** — Per-device topics. Devices publish to `devices/{id}/state`, `devices/{id}/telemetry`, `devices/{id}/logs`. Backend publishes to `devices/{id}/desired`, `devices/{id}/commands`.

**`flock/internal/v1/+`** — Internal topics. Consumed only by Flock backend services. Schema can evolve with a version bump and coordinated deployment. Examples: `flock/internal/v1/device/state`, `flock/internal/v1/build/status`.

**`flock/events/v1/+`** — Customer-facing topics. Stable, versioned, treated as a public API. The events gateway translates from internal topics to these — internal changes never break customer integrations. Bumping to v2 means running both versions in parallel until customers migrate. Examples: `flock/events/v1/device/online`, `flock/events/v1/rollout/completed`.

### Authentication

Fully delegated to **Zitadel**, self-hosted as a Helm subchart. The API validates tokens but never issues them for human users.

Zitadel handles email/password, social login, SAML, MFA, organization management, and enterprise SSO. Enterprise customers bring their own identity provider at the Zitadel level.

Devices use a separate lighter mechanism. On first boot a device presents a provisioning key to the API and receives a short-lived device-scoped JWT signed by Flock. This JWT is the credential for MQTT connections and registry pulls only.

Authorization uses RBAC: `owner`, `admin`, `developer`, `viewer` per organization. Enforced in API middleware, stored in PostgreSQL.

### Device Communication

**MQTT over TLS** between devices and the broker. Devices publish state, telemetry, logs, and events. Devices subscribe to desired state and command topics. Backend targets subscribe to device topics via shared subscription groups for horizontal scaling.

Synchronous internal communication between targets (tunnel management, build log streaming) uses **gRPC**.

### Release and Rollout Model

Three concepts with distinct responsibilities. This model handles the trivially simple case with no overhead and supports arbitrarily complex future rollout strategies without redesign.

**Release** — An immutable versioned artifact. A compose specification with all image references locked to specific digests. Created once, never modified. Has no inherent relationship to any fleet until a deployment references it.

**Deployment** — The intent to roll out a specific release to a specific set of devices under a specific strategy. A deployment defines:
- *Target release* — which release to roll out
- *Target selector* — which devices are eligible. Can be a whole fleet, or filtered by device labels, firmware version ranges, custom attributes, or any combination
- *Strategy* — how to proceed: immediate, canary (N% first), progressive (explicit stage percentages with configurable intervals), or manual (operator explicitly advances each stage)
- *Health criteria* — what signals determine a stage is healthy enough to advance: device online rate, OTA confirmation rate, custom telemetry thresholds from ClickHouse
- *Rollback policy* — conditions that trigger automatic rollback and which release to roll back to (defaults to previous release in deployment history if unspecified)

Creating a deployment does not immediately change anything on any device. It creates the plan.

**Rollout** — The live execution state of a deployment. Created by the scheduler when it begins executing a deployment. Tracks which devices have been targeted, which have confirmed the update, which have failed, and what stage the rollout is currently in. A fleet can have multiple concurrent rollouts if different device segments are targeted by different deployments. The scheduler drives rollouts forward and evaluates health criteria at each stage gate.

### Audit Logging

Every mutating action produces a structured audit event. Audit events are published to a dedicated internal MQTT topic and consumed by a writer that ships them to Loki as structured log streams labeled by org, actor, action type, and resource.

Operators can configure Loki retention per label selector and route audit streams to external SIEM systems via standard Loki output mechanisms.

### Customer Event Streaming

The `events-gateway` target subscribes to `flock/events/v1/+` MQTT topics and delivers to customer-configured subscriptions.

**Delivery mechanisms:**
- **Webhooks** — POST JSON to a customer HTTPS endpoint. HMAC signature verification. Automatic retry with exponential backoff. Delivery log queryable via API.
- **WebSocket** — Authenticated persistent connection. Events delivered in real time. Useful for customer dashboards and internal tooling.

Subscriptions are stored in PostgreSQL and filter by event type, fleet, or device. Customers receive only what they subscribed to.

**Event types include (non-exhaustive):** device registered/deregistered, device online/offline, deployment created, rollout stage advanced/completed/failed, OTA update completed/failed per device, build completed/failed, SSH session opened/closed, provisioning key created/revoked.

---

## flock-agent (flockd)

Single static Rust binary. Runs as a systemd service. No Node.js, no Python, no container runtime of its own. Compiled with musl libc.

Target architectures: `aarch64`, `armv7`, `armv6`, `x86_64` — Linux musl targets.

**Responsibilities:** MQTT connection to the broker, reconciling actual container state toward desired state, delta-aware image fetching, shipping telemetry and logs, managing the local WireGuard interface, self-updating, executing host commands on behalf of the platform (OS update orchestration).

**Delta-aware image fetching:** When updating a container, the agent checks whether a precomputed delta exists for the (old digest, new digest) pair. If yes, it fetches the delta from object storage, applies it locally against the cached base layer, and imports the reconstructed layer into the runtime. The device downloads only the diff — real bandwidth saving on constrained networks.

**Container runtime abstraction:** Speaks to runtimes via a trait interface. Implementations for containerd (gRPC) and Podman (REST). Configured per device. No Docker daemon required.

**Reconciler pattern:** Continuously compares desired state (last received via MQTT) against reported state (read from the runtime) and converges. Runs every 5 seconds.

**Offline resilience:** Backend unreachable means the device keeps running its last known desired state. State reports buffer locally and flush on reconnect.

**Registration:** On first boot reads a provisioning key from config, calls the API, receives credentials and WireGuard configuration, persists locally. Subsequent boots skip this.

### Host OS Update Orchestration

Flock is OS-agnostic — it does not own or build the host OS. Container workloads and the agent binary are managed directly; host OS updates are the operator's responsibility. Flock provides **orchestration** so operators can roll out OS updates safely using the same progressive deployment machinery used for container releases.

**How it works:** A deployment can target a **host command** instead of (or in addition to) a container release. The agent executes a user-provided script or command on the host, reports success or failure back to the platform, and the scheduler applies the same rollout strategies (canary, progressive, manual) and health criteria to decide whether to advance, pause, or rollback.

**What Flock does:**
- Distributes the command/script to targeted devices via the existing desired-state mechanism
- Executes it on the host as a controlled action with timeout and exit-code reporting
- Reports per-device success/failure to the scheduler
- Applies rollout strategy (canary %, stage gates, health checks) identically to container deployments
- Supports rollback by executing a separate rollback command if a stage fails

**What Flock does NOT do:**
- Build or distribute OS images
- Manage partitions, bootloaders, or A/B root filesystems
- Know anything about the host OS distribution or package manager

**Example use cases:**
- `apt-get update && apt-get upgrade -y` rolled out to 5% of devices, wait for health, continue
- A custom firmware flash script deployed progressively across a fleet
- A kernel parameter change applied with automatic rollback if devices go offline

This keeps Flock's "any OS" value proposition intact while giving operators production-grade rollout safety for host-level changes.

Implementation details defined in the flock-agent planning session.

---

## flock-ui

React SPA. Built to static assets, served by nginx in a container. No SSR, no server process. The right choice on Kubernetes — no framework server infrastructure, no platform-specific features, just files in a container.

**Dashboard:** organization and fleet management, device list with live status, device detail with metrics charts, log streaming, web terminal (xterm.js over WebSocket), release browser, deployment creation with full strategy configuration, rollout progress visualization, environment variable management, event subscription management.

**Documentation:** Co-located as static MDX content, built into the same nginx container at `/docs`.

Technology choices defined in the flock-ui planning session.

---

## flock-cli

Rust CLI binary (`flock`). Single static binary compiled with musl libc. The operator and developer interface to the platform.

Target architectures: `aarch64`, `x86_64` — Linux and macOS musl/native targets. Windows x86_64.

**Core commands:** `login`, `push`, `ssh`, `logs`, `devices`, `env`, `deploy`, `releases`, `fleets`.

Authenticates via Zitadel device authorization flow. Stores credentials locally. All commands talk to the `flock-api` REST API.

Implementation details defined in the flock-cli planning session.

---

## FlockOS

The thinnest possible layer on a standard Linux base. Not a custom OS like balenaOS — a minimal, opinionated Debian image that ships exactly what Flock needs and nothing else.

### Why debos, not Yocto

Yocto is too slow to iterate on, has a steep learning curve, and its BitBake DSL is unfriendly to LLM-assisted development. debos uses declarative YAML recipes on top of Debian's package ecosystem — faster builds, easier contributions, standard tooling.

### Base

Debian Bookworm. Read-only root filesystem with A/B partitions for atomic OS updates and automatic fallback.

### Contents

containerd, flock-init, WireGuard, cloud-init. No desktop environment, no package manager on the running system, no SSH server by default (SSH access is through the Flock tunnel).

### A/B Partition Scheme

Two root partitions. The active partition is mounted read-only. Updates write to the inactive partition, update the bootloader to point to it, and reboot. If the new partition fails to boot (no heartbeat within timeout), the bootloader falls back to the previous partition automatically.

### Official Images

- `aarch64-generic` — Raspberry Pi 4/5, generic arm64 SBCs
- `amd64-generic` — Intel/AMD x86_64 devices
- `armv7-generic` — Older 32-bit ARM boards
- `arm64-nvidia` — NVIDIA Jetson (Orin, Xavier) with JetPack runtime libraries

### Community Images

Contributed via PR to the `flock-os` repo. Each community recipe lives in its own directory with a debos YAML recipe and any overlay files. CI builds and publishes images on merge. Community maintainers own their recipes.

---

## Device Provisioning

Three paths for getting devices connected to the platform.

**Dashboard download (zero-friction):** Configure fleet and WiFi credentials in the UI, download a FlockOS image with an embedded provisioning key and cloud-init config. Flash to SD card or eMMC, plug in, device boots and registers automatically.

**CLI configure (existing OS):** `flock configure-device --fleet my-fleet` SSHes into an existing Linux device, installs flock-init and containerd, writes the Flock config, and starts the service. For operators who already have their own OS and just want the agent.

**curl installer (quick start):** `curl -fsSL https://get.flock.io | bash -s -- --fleet-key <key>` for existing Linux devices. Downloads and installs flock-init and containerd, registers the device. Fastest path for experimentation.

---

## flock-base

Helm chart that deploys the full Flock platform to any Kubernetes cluster with `helm install`.

**Infrastructure subcharts** (each disableable to use external managed services): PostgreSQL, EMQX, Valkey, ClickHouse, Loki, Harbor, MinIO, Zitadel.

**Deployment modes:**
- *Single-binary* (`singleBinary: true`): all targets in one Deployment. For staging and small self-hosted.
- *Distributed*: each target its own Deployment with independent HPA. For production.

`builder` and `delta` scale to zero when idle. `tunnel` requires a LoadBalancer service for external traffic. EMQX requires a LoadBalancer service for device MQTT connections.

---

## flock-staging / flock-production

Each contains only Helm values overrides and a GitHub Actions workflow. On push to main the workflow runs `helm upgrade --install` against the target cluster. Secrets encrypted with Helm Secrets using age encryption, stored in the repo.

Staging uses single-binary mode. Production uses distributed mode.

---

## Free Deployment (Development Phase)

**Backend:** Fly.io. Single Machine running `--target=all`. Fly Managed Postgres. Zitadel on a second Fly Machine. EMQX on a third Fly Machine.

**External free tiers:**
- Upstash Valkey for KV state
- ClickHouse Cloud for telemetry
- Cloudflare R2 for object storage
- Grafana Cloud for Loki (audit + device logs)

**UI:** Cloudflare Pages. Auto-deploys from GitHub.

**Scaling path:** Hetzner VPS (~€5/month, 4GB RAM) running k3s with the full Helm chart when free tiers are outgrown.

---

## Build Phases

Implementation order. Each phase is broken into concrete tasks in a separate planning session before work begins on that phase.

1. **Foundation** — `flock-api` repo, multi-target binary, config, infrastructure connections, docker-compose, health check.
2. **Identity and Auth** — PostgreSQL schema, Zitadel, JWT validation, RBAC, provisioning keys, audit event publishing.
3. **Core API** — All REST handlers. OpenAPI spec committed.
4. **Device Connectivity** — MQTT broker, ingester, live device state, internal event publishing.
5. **Rollout Engine** — Scheduler, deployment execution, all strategies, health criteria, auto-rollback.
6. **Build Pipeline** — Builder, multi-arch builds, log streaming.
7. **Registry and Deltas** — Harbor subchart, registry-proxy, delta generation and serving.
8. **Remote Access** — Tunnel, WireGuard overlay, SSH proxy, public device URLs.
9. **Event Streaming** — Events gateway, versioned topics live, webhook and WebSocket delivery, customer subscriptions.
10. **flock-agent** — Rust agent, real device connects, OTA updates, SSH, telemetry.
11. **flock-ui** — React dashboard, full workflow, rollout visualization, web terminal.
12. **CLI** — Rust CLI, core commands: `login`, `push`, `ssh`, `logs`, `devices`, `env`.
13. **Helm and Deployment** — `flock-base` complete, staging deploys via GitHub Actions, self-hosting guide.
14. **FlockOS** — debos recipes, A/B update logic, cloud-init integration, first-boot provisioning, official image builds in CI.

---

## Open Decisions

To be resolved before the relevant phase.

- **Rollout health signals (before Phase 5):** Minimum viable signal set for stage gate evaluation. Recommendation: device online rate + OTA confirmation rate as built-ins, with an optional custom ClickHouse query threshold that operators define in the deployment spec.
- **Delta algorithm (before Phase 7):** bsdiff is the working assumption. Evaluate zstd-based diffing before committing to implementation.
- **Agent self-update (before Phase 10):** Agent manages its own binary update via a staging path and signals systemd, or a privileged sidecar container manages the agent binary.
- **Host command schema (before Phase 10):** Define the deployment spec extension for host commands — command string, timeout, expected exit code, rollback command. The scheduler and agent already support the orchestration model; this decision is about the exact API surface.