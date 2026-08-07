# RBAC analysis: what would it take to actually run `plan_mustgather`?

**Status: exploratory analysis only. No RBAC or `config.toml` changes have been
made as a result of this doc, and none should be without a deliberate,
separate decision — same rule as everywhere else in this repo (see
CLAUDE.md "Known limitations" and Security notes).** This exists to answer
the question "what would the `claude-code` identity need beyond `ClusterRole` `view` to
run must-gather end-to-end", triggered by actually running the
`plan_mustgather` prompt (2026-08-06) and hitting three `resources_create_or_update`
permission warnings (Namespace, ServiceAccount, ClusterRoleBinding, Pod all
blocked).

## Two different permission questions, easy to conflate

The must-gather plan involves **two separate identities** with **two
separate permission needs**:

1. **The caller** — whoever creates the four bootstrap objects (Namespace,
   ServiceAccount, ClusterRoleBinding, Pod). Today that's meant to be a
   human with `oc apply`, not the `claude-code` SA — `read_only` and
   `--disable-destructive` block it deliberately (SETUP.md §2 Option B).
2. **The must-gather pod's own ServiceAccount** (`must-gather-collector`
   in the generated plan) — whatever *it* can do determines what the
   gather script can actually collect once running.

Widening `claude-code`'s RBAC only affects (1). It has no bearing on (2) —
the plan hard-codes the pod's ServiceAccount to a `ClusterRoleBinding`
naming `cluster-admin` regardless of what `claude-code` itself holds
(assuming it could even create that binding — see below).

## What (1) would need, mechanically

To let `claude-code` apply the exact plan the prompt generated, its
ClusterRole would need, at minimum:

```yaml
- apiGroups: [""]
  resources: ["namespaces"]
  verbs: ["create", "get", "delete"]
- apiGroups: [""]
  resources: ["serviceaccounts"]
  verbs: ["create", "get", "delete"]
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["create", "get", "delete"]
- apiGroups: [""]
  resources: ["pods/log"]
  verbs: ["get"]
- apiGroups: ["rbac.authorization.k8s.io"]
  resources: ["clusterrolebindings"]
  verbs: ["create", "get", "delete"]
```

That list looks tame. It isn't — the last rule is where this stops being a
small ask.

## The actual blocker: Kubernetes' escalation check

Kubernetes RBAC has a built-in rule (not OpenShift-specific, not this
cluster's `view` role's doing — it's core `rbac.authorization.k8s.io`
behavior): **an identity cannot create a `RoleBinding`/`ClusterRoleBinding`
that grants permissions it does not itself already hold**, unless it also
has the `escalate` verb on `clusterroles`/`clusterrolebindings`, or is
already `cluster-admin`. This is the platform's defense against exactly
the privilege-escalation path this plan would otherwise open (a `view`-only
identity minting itself a path to `cluster-admin` by proxy, via a
ServiceAccount it controls).

Concretely: giving `claude-code` `create` on `clusterrolebindings` alone
would **not** let it apply this plan's `ClusterRoleBinding` (`roleRef:
cluster-admin`) — the API server would reject it with something like
`attempting to grant RBAC permissions not currently held`. To make that
`oc apply` succeed, `claude-code` would need to *already* hold
cluster-admin-equivalent permissions itself, or hold the `escalate` verb
(which is functionally the same trapdoor by another name).

**I have not verified this live** — it can't be tested without first
granting the write access under discussion, which is the thing this doc is
trying to inform, not assume. It's stated here as documented core
Kubernetes RBAC behavior, not as a live-probed fact of this cluster (unlike
the Node/MachineConfigPool `Forbidden` results in CLAUDE.md, which *are*
live-confirmed).

**Bottom line:** there is no "small, additional grant" that lets
`claude-code` apply *this exact plan* (cluster-admin ClusterRoleBinding
included) while staying meaningfully short of cluster-admin itself. Any
RBAC change that makes the plan applicable is, in substance, a decision to
let this identity mint cluster-admin access on demand — which conflicts
directly with why this repo's `view`-only ClusterRoleBinding exists in the
first place (CLAUDE.md: "this instance needs cluster-wide read access for
investigations", not write, and Security notes' insistence on rotating
short-lived tokens rather than standing broad access).

## Why the pod's own SA defaults to `cluster-admin` (question 2)

Separately from the above, it's worth understanding *why* Red Hat's own
`oc adm must-gather` (which this prompt mirrors) defaults the collector
pod's ServiceAccount to `cluster-admin` rather than something narrower:

- It reads cluster-scoped config (`ClusterVersion`, `ClusterOperator`,
  `MachineConfigPool`, etc.) — some of which, per CLAUDE.md, `view` itself
  can't reach on this cluster.
- Several must-gather plugins create per-node debug pods (`oc debug node/
  ...` equivalent) to pull host-level logs and config, which needs Node
  access plus the ability to run privileged pods — well beyond `view`.
- It gathers across *every* namespace uniformly, including operator/
  platform namespaces with tighter default RBAC than user namespaces.
- Red Hat support tooling is built assuming a "complete" must-gather
  bundle; a deliberately trimmed one risks being rejected or requiring a
  re-run mid-case, which defeats the point of gathering proactively.

## If a reduced-scope collector were ever wanted anyway

This is a sketch, not a recommendation — flagging it because it's the
natural follow-up question, but it would need real validation before
trusting it for a support case:

- Swap the plan's `ClusterRoleBinding` `roleRef` from `cluster-admin` to a
  **new custom ClusterRole**, scoped to what most gather plugins actually
  read: `view`'s existing scope, plus `nodes`/`nodes/log` get-list-watch,
  `machineconfigpools`/`machineconfigs` get-list, and read on
  `securitycontextconstraints`.
- This would very likely **not** include per-node debug-pod collection
  (needs a privileged SCC + node exec, which is most of what makes
  must-gather's default so broad) — so the output would be a reduced
  bundle, not a drop-in replacement for the real thing.
- Untested. Would need a real must-gather run against a non-prod cluster,
  diffed against a `cluster-admin`-collected bundle, to know what's
  actually missing before anyone relies on it for a real support case.

## Recommendation

Leave `claude-code` on `view`. The practical pattern for actually running
must-gather stays: `plan_mustgather` generates the manifests safely
read-only (as it does today), and a human applies/cleans them up with
their *own*, separately-privileged `oc login` — which is also how
`oc adm must-gather` is normally run by a cluster admin anyway, not
automated. That keeps this repo's standing RBAC at `view` (Security notes:
short-lived, minimal, rotated) while still allowing must-gather to happen
on demand, just not through this MCP identity.
