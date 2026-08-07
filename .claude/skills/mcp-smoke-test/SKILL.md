---
name: mcp-smoke-test
description: End-to-end smoke test for kubernetes-mcp-server — re-verifies the RBAC/config handshake, then actually drives the live MCP tools and cross-checks their output against `oc` ground truth. Run after regenerating the token/kubeconfig, after changing config.toml or the RBAC binding, or before trusting the server for a real investigation.
---

# kubernetes-mcp-server smoke test

Never assume the handshake still holds from a previous run — the token is
short-lived (8h convention, CLAUDE.md) and RBAC/config can drift. Re-verify
first, then actually drive the tools instead of trusting `claude mcp list`
alone.

## 1. Re-check the handshake

Cheap checks, run these first — fix per SETUP.md §2/§8 and stop before
continuing to step 2 if any fail:

```bash
export KUBECONFIG=~/claude/openshift-mcp-server.kubeconfig   # do NOT unset/merge into ~/.kube/config

# Credential present and token not expired (empty `user: {}` fails silently otherwise)
oc whoami                                    # expect: system:serviceaccount:cluster-ops:claude-code
oc get ns --request-timeout=5s | head -3     # expect: succeeds, no 401/Unauthorized/token-expired error

# Secrets still denied at the RBAC layer, independent of the MCP config
oc auth can-i get secrets --as=system:serviceaccount:cluster-ops:claude-code --all-namespaces   # expect: no

# kubeconfig permissions
stat -c '%a' ~/claude/openshift-mcp-server.kubeconfig   # expect: 600

# MCP connection itself
claude mcp list   # expect: kubernetes-mcp-server ... ✔ Connected
```

Also confirm which RBAC binding is actually live (this repo currently applies
the cluster-wide `ClusterRoleBinding`, per CLAUDE.md — don't assume it's
still Option B without checking):

```bash
oc get clusterrolebinding claude-code-crb -o yaml 2>/dev/null || oc get rolebinding claude-code-rb -n cluster-ops-o yaml 2>/dev/null
```

## 2. Drive the live MCP tools and cross-check against ground truth

This is the step the handshake checks above don't cover: they prove the
*connection* works, not that the *tools* return real, correctly-scoped data.
Use the `mcp__kubernetes-mcp-server__*` tools directly (this session has them
available) — don't just re-run `oc` commands.

1. **List pods in `cluster-ops`** via `pods_list_in_namespace` (namespace:
   `cluster-ops`). Independently run
   `oc get pods -n cluster-ops --kubeconfig=~/claude/openshift-mcp-server.kubeconfig`
   and confirm the pod names/count match exactly. This is the check that the
   tool is returning live cluster state, not a cached or hallucinated
   response. (An empty result is a valid match if `cluster-ops` genuinely has
   no workloads — don't mistake "no pods" for "tool is broken".)

2. **Confirm the live scope matches the binding found in step 1.** With the
   cluster-wide `ClusterRoleBinding` currently applied, `pods_list_in_namespace`
   should succeed for namespaces *other than* `cluster-ops` too (e.g. try one
   more namespace that's known to have pods) — that's expected, not a leak,
   given Option B was a deliberate choice (CLAUDE.md). If this repo is ever
   switched to a namespaced `RoleBinding` (Option A), redo this check
   expecting the opposite: success only in the granted namespace, and a
   permission error everywhere else.

3. **Confirm `denied_resources` blocks Secrets at the MCP layer**, not just
   RBAC. Call `resources_list` with `apiVersion: v1`, `kind: Secret` (any
   namespace) — expect an explicit `resource not allowed` error from the MCP
   server itself, distinct from an RBAC `Forbidden`. This is the specific
   gap CLAUDE.md's security notes call out: `read_only` alone does not block
   Secret *reads*, only `denied_resources` does.

4. **Confirm no destructive tool is exposed.** Check the current MCP tool
   list for this server (e.g. via `ToolSearch` or the session's tool
   listing) and confirm there's no delete/create/patch-capable tool. Given
   `toolsets = ["core", "config", "kubevirt", "openshift"]` and
   `--disable-destructive`/`read_only = true`, the expected tool set
   (confirmed live 2026-08-06 on the Tech Preview image, tag `0.4`) is:
   `events_list`, `namespaces_list`, `nodes_log`, `nodes_stats_summary`,
   `nodes_top`, `pods_get`, `pods_list`, `pods_list_in_namespace`,
   `pods_log`, `pods_top`, `projects_list`, `resources_get`,
   `resources_list`, `vm_guest_info`. **`configuration_view` is expected to
   be absent** on this image despite `config` being enabled — that's a
   known gap versus the community npm build (CLAUDE.md), not a
   misconfiguration; don't flag it as a finding. Neither `kubevirt` nor
   `openshift` should add any write-capable tool (`vm_create`,
   `vm_lifecycle`, `vm_clone` must NOT appear) — `openshift` only adds the
   `plan_mustgather` *prompt* (manifest generation, not a callable tool), so
   it shouldn't show up in this tool-list check at all. Toolsets can still be
   widened by mistake — this step catches that directly instead of trusting
   the flag.

5. **Confirm the custom and restored prompts are actually registered.**
   Check `prompts/list` (or the client's prompt listing) for all five
   expected entries: `cluster-health-check`,
   `openshift-virtualization-troubleshooting` (this repo's custom prompt),
   `plan_mustgather`, `vm-troubleshoot`, `windows-golden-image`. Missing
   `plan_mustgather` or `vm-troubleshoot`/`windows-golden-image` most likely
   means the registration silently fell back to a different image or the
   community `npx` path — re-check `claude mcp list`'s exact invocation
   against SETUP.md §3 (podman command, image tag, `--userns=keep-id`).

## 3. Report

Summarize as a pass/fail table: handshake (whoami / token liveness / secrets
RBAC / kubeconfig perms / mcp connected) and live-tool checks (pods match
ground truth / scope matches the applied binding / Secret read blocked by
config / no destructive tool exposed). Flag anything that's expected-but-
easy-to-misread as a regression (e.g. cluster-wide access succeeding is
correct under Option B) so it isn't reported as a finding.
