# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

This is not application source code — it's the local configuration and RBAC
artifacts for running `kubernetes-mcp-server` (upstream:
https://github.com/openshift/openshift-mcp-server) against an OpenShift
cluster from Claude Code. There is no build, lint, or test tooling; changes
here are cluster credentials and access-control manifests, not code.

[docs/SETUP.md](docs/SETUP.md) is the single source of truth for how this is *supposed*
to be configured, and is written as a hardened, least-privilege setup guide.
**Treat SETUP.md as the spec — when editing the YAML/shell files in this
directory, check them against it, and when SETUP.md changes, update the
generated artifacts (or flag the drift) rather than letting them diverge.**

## Files in this directory

- `templates/serviceaccount.yaml` / `templates/namespace.yaml` /
  `templates/clusterrolebinding.yaml` — the RBAC identity the MCP server
  authenticates as (`claude-code` SA in `cluster-ops`, bound to the `view`
  ClusterRole). Applied with `oc apply -f templates/<file>.yaml`. It's a
  `ClusterRoleBinding`, not a namespaced `RoleBinding` — that's deliberate
  (SETUP.md §2 Option B): this instance needs cluster-wide read access for
  investigations, not a single namespace. Don't "fix" it back to Option A
  without checking with the user first.
- `config.toml` — the hardened MCP server config (SETUP.md §4): `read_only`,
  `denied_resources` blocking `Secret`, and a trimmed `toolsets` list. This
  is the live config, not a template — it's referenced directly via
  `--config` in the registered MCP server command (`claude mcp list` shows
  the exact invocation, including `--disable-destructive` and
  `--disable-multi-cluster`). Edit it in place.
- There is no checked-in `cmd` script or `token` file in this directory —
  token minting and kubeconfig building follow SETUP.md §2 directly (run ad
  hoc, not scripted here). If you save a token to a standalone file for
  convenience, `chmod 600` it immediately and delete it once it's no longer
  needed (see Security notes below).

## Security notes specific to this repo

- Every file that can contain a live token or key
  (any standalone token file, `~/claude/openshift-mcp-server.kubeconfig`)
  must be `chmod 600`. Check this after regenerating either — `oc create
  token` and `oc config set-credentials` don't set restrictive permissions
  themselves.
- Token `--duration` should be short-lived and rotated, not a multi-month/year
  value minted once for convenience (SETUP.md §1.4, §2 "Token renewal").
  Working convention here is `8h` — long enough for a session, short enough
  to force rotation; there's no automated renewal set up, so a new token has
  to be minted and the kubeconfig updated once it expires.
- Don't reintroduce `--insecure-skip-tls-verify` when building a kubeconfig —
  SETUP.md §2 documents extracting the real CA cert via
  `oc config view --minify --raw` specifically to avoid needing that flag.
- `denied_resources` in `config.toml` must always block `Secret` reads, even
  with `read_only = true` — read-only mode still permits reading Secret
  *values*, only blocks writes. Verify with
  `oc auth can-i get secrets --as=system:serviceaccount:cluster-ops:claude-code --all-namespaces`
  (expect `no`) after any RBAC change, not just a config change.
- This directory's kubeconfig must stay isolated from `~/.kube/config` and
  never get merged into or used to overwrite the user's personal/admin
  kubeconfig.

## Known limitations of the current RBAC/config (observed, not spec)

These aren't things to "fix" unilaterally — they're facts about how this
cluster's `view` ClusterRole and MCP config actually behave in practice,
useful context before starting an investigation:

- **Node reads are forbidden**, despite the cluster-wide `ClusterRoleBinding`
  to `view` (SETUP.md §2 Option B). `resources_get`/`resources_list` on
  `v1 Node` returns `Forbidden` — this cluster's `view` role doesn't include
  node gets at cluster scope. Confirmed 2026-08-06.
- **The metrics API isn't reachable** — `nodes_top` and `pods_top` fail with
  `metrics API is not available` (no metrics-server/Prometheus adapter
  wired up for this path). Don't rely on these tools for resource-usage
  investigations (e.g. checking for memory pressure); there's currently no
  substitute available through this MCP server.
- **`events_list` only covers a ~1h window** — Kubernetes Events expire on
  their default TTL, so an empty/negative result (e.g. "no OOM events")
  only rules out the recent window, not historical occurrences. For
  anything older, this MCP server has no equivalent — it would require
  cluster logging/Prometheus access outside this setup.

If any of these need to change (e.g. granting node reads, wiring up
metrics), that's a deliberate RBAC/config decision — check with the user
first, same as the ClusterRoleBinding scope above.

## Skills

- `.claude/skills/mcp-smoke-test/` — end-to-end smoke test for
  `kubernetes-mcp-server`. Re-checks the handshake (token liveness, Secrets
  denied, kubeconfig has a credential, `claude mcp list` connected) and then
  actually drives the live MCP tools (pods, resources) and cross-checks
  their output against `oc` ground truth, rather than trusting the
  handshake alone. Run it after regenerating the token/kubeconfig, changing
  `config.toml`, or changing the RBAC binding.
