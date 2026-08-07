# Using this MCP server: available prompts and worked examples

This is the single reference for *how to actually use* the `kubernetes-mcp-server`
registered for this project — what prompts are available and how to invoke
them, plus a worked example of running a real investigation through MCP
tool calls only (no `oc`/`kubectl`/shell touching the cluster). Previously
split across separate docs with no cross-reference; merged here so there's
one place to look instead of several partial ones.

Every result below is a **live call** made through the MCP server actually
registered for this project (see `claude mcp list`), captured 2026-08-07
against the Red Hat Tech Preview image (`registry.redhat.io/openshift-mcp-tech-preview/openshift-mcp-server-rhel9:0.4`,
see `docs/SETUP.md` §3) — not a description of a hypothetical path.

---

## Part 1 — Available prompts

MCP servers can expose "prompts": pre-built workflows the client (Claude
Code) surfaces as slash commands, in the form `/mcp__<servername>__<promptname>`
([Claude Code docs](https://code.claude.com/docs/en/mcp.md#use-mcp-prompts-as-commands)).
Type `/` in Claude Code to see the exact resolved command for this server —
server/prompt names get normalized (spaces become underscores; hyphen
handling isn't documented precisely enough to promise an exact copy-paste
string here), so treat the forms below as the pattern, and confirm the
literal string via `/` before relying on it.

This repo's `config.toml` enables `core`, `config`, `kubevirt`, and
`openshift` (SETUP.md §4), which together provide five prompts:

| Prompt | Source | Arguments | What it does |
|---|---|---|---|
| `cluster-health-check` | built-in, `core` (always available) | `namespace` (optional, default all namespaces), `check_events` (optional, `true`/`false`, default `true`) | Comprehensive health assessment: nodes, cluster operators (OpenShift only), pods, workload controllers, PVCs, recent events. |
| `plan_mustgather` | built-in, `openshift` toolset | All optional: `node_name`, `node_selector`, `source_dir`, `namespace`, `gather_command`, `timeout`, `since`, `host_network`, `keep_resources`, `all_component_images`, `images` | Generates YAML manifests for a must-gather run (temp namespace + ServiceAccount + **cluster-admin** ClusterRoleBinding + privileged pod). `read_only`/`--disable-destructive` block this identity from ever applying what it plans. |
| `vm-troubleshoot` | built-in, `kubevirt` toolset | `namespace` (required), `name` (required) | Step-by-step OpenShift Virtualization VM troubleshooting guide: VM/VMI status, volumes, virt-launcher pod + logs, related events. |
| `windows-golden-image` | built-in, `kubevirt` toolset | `winImageDownloadURL` (required), `namespace`/`windowsVersion`/`pipelineVersion` (optional) | Builds a Windows golden image via a Tekton pipeline. **Not usable as configured here** — needs the `tekton` toolset too, which this repo doesn't enable. Listed for completeness; enabling it is a deliberate scope decision (check with the user first, per CLAUDE.md). |
| `openshift-virtualization-troubleshooting` | custom, this repo's `config.toml` | `vm_name` (required), `namespace` (required) | Wraps VM troubleshooting with this cluster's specific gaps (no Node reads, no metrics API, ~1h event window) so a clean result isn't misread as "all clear". See CLAUDE.md for why it exists alongside the built-in `vm-troubleshoot`. |

Example invocations (verify the exact slash-command string via `/` first):

```
/mcp__kubernetes-mcp-server__cluster-health-check
/mcp__kubernetes-mcp-server__cluster-health-check namespace=cluster-ops check_events=false
/mcp__kubernetes-mcp-server__vm-troubleshoot namespace=openshift-cnv name=my-vm
/mcp__kubernetes-mcp-server__openshift-virtualization-troubleshooting vm_name=my-vm namespace=openshift-cnv
```

You don't have to use the slash form — asking in natural language ("check
the health of my cluster", "why won't VM my-vm start in namespace foo")
works too; Claude will call the same prompt or the underlying tools
directly depending on what's registered and what fits the request.

---

## Part 2 — Worked example: checking for OOM evidence, cluster-wide and namespace-scoped

### Primary tool: `events_list`

`mcp__kubernetes-mcp-server__events_list` lists Kubernetes Events. Omitting
`namespace` queries **all namespaces** in one call; passing `namespace`
scopes it to one. `fieldSelector` narrows either form server-side.

**Cluster-wide, OOM-specific:**

```
events_list(fieldSelector: "reason=OOMKilling")
```

Live result: `# No events found`

**Namespace-scoped, OOM-specific** (using `cluster-ops`, this identity's
own namespace):

```
events_list(namespace: "cluster-ops", fieldSelector: "reason=OOMKilling")
```

Live result: `# No events found`

**When to scope to a namespace vs. go cluster-wide:** go cluster-wide when
you don't yet know which namespace is affected (a broad sweep); scope to a
namespace once you already suspect a specific workload — it's the same
tool, just a narrower, cheaper, less noisy query. Both forms hit the same
1-hour event TTL limitation (below), so neither one "sees further back"
than the other — the namespace argument only narrows *what*, not *how far
back*.

**Cluster-wide, broad sweep** (proves the call reaches real, varied
cluster data, not a canned/empty stub):

```
events_list(fieldSelector: "type=Warning")
```

Live result: 13 Warning events across several namespaces (`argocd`,
`default`, others) — mostly `ExternalSecret`/`ClusterSecretStore`
credential-resolution failures and an SCC-related pod admission failure.
None had reason `OOMKilling` or `OOMKilled`.

### The catch: events expire

Per CLAUDE.md ("Known limitations"), Kubernetes Events have a short TTL —
this cluster's window is roughly **1 hour**. A `No events found` result
only rules out OOMs in that recent window, not historically, regardless of
whether the query was cluster-wide or namespace-scoped. There is no MCP
tool here that queries older event history (that would need cluster
logging/Prometheus, out of scope for this setup).

### Corroborating check that outlives the event TTL

A container that was OOM-killed leaves a mark on the Pod object itself —
`status.containerStatuses[].lastState.terminated.reason == "OOMKilled"` —
which persists until the pod is deleted or replaced, well past the 1h event
window.

**Important, confirmed 2026-08-07 (this corrects the original version of
this doc):** `resources_list` returns a flattened **table** by default
(`NAMESPACE APIVERSION KIND NAME READY STATUS RESTARTS AGE ...`), not full
YAML — the `containerStatuses[].lastState` field simply isn't in that
output. `resources_get` (singular, one named object) always returns full
structured detail including `containerStatuses`, confirmed live against a
real pod. So the corroboration workflow is necessarily two steps, not one:

1. **List candidates** (table output is fine here — just need `RESTARTS` >
   0 as a filter), cluster-wide or namespace-scoped:
   ```
   resources_list(apiVersion: "v1", kind: "Pod")                          # cluster-wide
   resources_list(apiVersion: "v1", kind: "Pod", namespace: "cluster-ops") # namespace-scoped
   ```
   Live cluster-wide result: pods returned in table form, e.g.
   `argocd-application-controller-0` with `RESTARTS 3`. Live
   `cluster-ops`-scoped result: empty (this namespace currently has no
   pods — a valid result, not a broken tool; don't mistake "no pods" for
   "tool is broken").

2. **Inspect each restart-count candidate individually** to get the actual
   last-termination reason:
   ```
   resources_get(apiVersion: "v1", kind: "Pod", namespace: "argocd", name: "argocd-application-controller-0")
   ```
   Live result: `lastState: {}` for the container with 3 restarts — empty,
   not `OOMKilled`. (An empty `lastState` here means no abnormal
   termination is currently retained for that container, which is itself
   an answer, not a missing one.)

Still zero `oc`/`kubectl` — just `resources_list` to find candidates plus
`resources_get` per candidate to confirm or rule out `OOMKilled`.

### What's explicitly NOT part of this path

- `pods_top` / `nodes_top` — fail on this cluster (no metrics API wired
  up), so they can't corroborate with live memory-pressure numbers.
  Documented already in CLAUDE.md.
- Any shell, `oc`, or `kubectl` invocation. Every query above was executed
  as a direct MCP tool call.

### Recommended cluster-wide OOM check, end to end

1. `events_list(fieldSelector: "reason=OOMKilling")` — fast, cheap, catches
   anything in the last ~1h across the whole cluster. Add `namespace: "..."`
   to scope it once a suspect namespace is known.
2. If historical/older evidence is needed, `resources_list(apiVersion:
   "v1", kind: "Pod")` (optionally namespace-scoped) to find restart-count
   candidates, then `resources_get` on each candidate to check
   `status.containerStatuses[].lastState.terminated.reason`.
3. Treat a clean result from both as "no OOM evidence in the checkable
   window/scope," not "no OOM ever" — say so explicitly, don't overclaim.
