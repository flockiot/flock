# flock

Umbrella repository for the Flock device fleet management platform. All component repos are git submodules.

## Architecture

See `arch.md` for the full platform specification.

## Submodules

- **flock-api** — Go backend (multi-target binary)
- **flock-agent** — Rust on-device agent
- **flock-cli** — Rust CLI
- **flock-ui** — React SPA dashboard
- **flock-base** — Helm base chart
- **flock-local** — Local dev Helm overrides (Docker Desktop k8s)

## Clone

```bash
git clone --recurse-submodules git@github.com:flockiot/flock.git
```

## Update All Submodules

```bash
git submodule update --remote --merge
```

## Conventions

- Tests are part of every task, not a follow-up
- No stubs or scaffolding unless explicitly asked
- Prefer explicit and readable over clever
- Work in small, verifiable increments
- When a decision is not in arch.md, ask before assuming

## Code Style

- Do NOT write comments that explain what the code does — the code should speak for itself
- Only write comments when explaining **why** something non-obvious is done, or linking to an issue/spec
- No redundant/repetitive comments

## Git Workflow

- Each task gets its own branch off `main`
- Use **conventional commits**: `feat:`, `fix:`, `chore:`, `docs:`, `test:`, `ci:`
- Commit message must reference the GitHub issue: `Closes #N`
- One commit per PR, squash if needed
- Open PR with `gh pr create` and add `--reviewer SyntaxSmith106`
