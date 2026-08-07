# Checking for cluster-wide OOMs via the MCP server (not CLI)

This documents how to ask `kubernetes-mcp-server` for cluster-wide OOM
evidence using only MCP tool calls — no `oc`, `kubectl`, or shell commands
touch the cluster. Every result below is a live call made through the MCP
server registered for this project (see `claude mcp list`), captured
2026-08-06 as proof the path actually works, not a description of a
hypothetical one.

## Primary tool: `events_list`

`mcp__kubernetes-mcp-server__events_list` lists Kubernetes Events. Omitting
`namespace` queries **all namespaces** in one call — that's the cluster-wide
part. `fieldSelector` narrows it server-side.

### Query 1 — broad sweep

```
events_list(fieldSelector: "type=Warning")
```

Live result (2026-08-06 19:0x CDT): 60+ Warning events returned, spread
across a dozen-plus namespaces — probe failures, `FailedMount`, `BackOff`,
operator resolution conflicts. None had reason `OOMKilling` or `OOMKilled`.
This confirms the call reaches the real cluster and returns real, varied
data — not a canned or empty stub.

### Query 2 — OOM-specific

```
events_list(fieldSelector: "reason=OOMKilling")
```

Live result: `# No events found`

That's the direct answer to "any OOMs cluster-wide right now?" — a clean,
specific, server-filtered query, answered with real output, using one MCP
tool call.

## The catch: events expire

Per this repo's `CLAUDE.md` ("Known limitations"), Kubernetes Events have a
short TTL — this cluster's window is roughly **1 hour**. A `No events found`
result only rules out OOMs in that recent window, not historically. There is
no MCP tool here that queries older event history (that would need cluster
logging/Prometheus, which is out of scope for this setup).

## Corroborating check that outlives the event TTL

A container that was OOM-killed leaves a mark on the Pod object itself —
`status.containerStatuses[].lastState.terminated.reason == "OOMKilled"` —
which persists until the pod is deleted or replaced, well past the 1h event
window. There's no server-side field selector for that nested path, so the
check is: list pods cluster-wide and read `containerStatuses` in the
returned YAML.

```
resources_list(apiVersion: "v1", kind: "Pod")   # no namespace = all namespaces
```

Then scan each pod's `status.containerStatuses[].lastState.terminated.reason`
for `OOMKilled`. Still zero `oc`/`kubectl` — just a second MCP tool call plus
reading the structured response.

## What's explicitly NOT part of this path

- `pods_top` / `nodes_top` — fail on this cluster (no metrics API wired up),
  so they can't corroborate with live memory-pressure numbers. Documented
  already in `CLAUDE.md`.
- Any shell, `oc`, or `kubectl` invocation. Both queries above were executed
  as direct MCP tool calls in this session.

## Recommended cluster-wide OOM check, end to end

1. `events_list(fieldSelector: "reason=OOMKilling")` — fast, cheap, catches
   anything in the last ~1h.
2. If historical/older evidence is needed, `resources_list(apiVersion: "v1",
   kind: "Pod")` and scan `lastState.terminated.reason` across all
   namespaces.
3. Treat a clean result from both as "no OOM evidence in the checkable
   window," not "no OOM ever" — say so explicitly, don't overclaim.
