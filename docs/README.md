# docs/

Supporting documentation for this repo's `kubernetes-mcp-server` setup.
Start with [`SETUP.md`](SETUP.md) — everything else builds on it.

- **[SETUP.md](SETUP.md)** — the hardened, least-privilege setup guide.
  Source of truth for how the ServiceAccount, RBAC binding, token, kubeconfig,
  and `config.toml` in this repo are *supposed* to be configured. Read this
  first if you're standing up a fresh instance or auditing an existing one.
- **[MCP-OOM-CHECK.md](MCP-OOM-CHECK.md)** — a worked example of asking the
  MCP server for cluster-wide OOM evidence using only MCP tool calls (no
  `oc`/`kubectl`/shell), with real captured output proving the path works.

## Official upstream documentation

This repo only holds *this environment's* config and RBAC artifacts — the
server itself, its full tool reference, and its configuration options are
documented upstream at
[openshift/openshift-mcp-server](https://github.com/openshift/openshift-mcp-server):

- [Tools and Functionalities](https://github.com/openshift/openshift-mcp-server#tools-and-functionalities) — the full list of every MCP tool (command) `kubernetes-mcp-server@latest` exposes, grouped by toolset (`core`, `config`, `openshift`, `helm`, `kubevirt`, `tekton`, etc.), plus which toolsets are enabled by default.
- [Configuration reference](https://github.com/openshift/openshift-mcp-server/blob/main/docs/configuration.md) — every `config.toml` option and CLI flag (`--read-only`, `--disable-destructive`, `--toolsets`, `--config`, ...).
- [Getting started with Claude Code](https://github.com/openshift/openshift-mcp-server/blob/main/docs/getting-started-claude-code.md) — upstream's own Claude Code integration guide.
- [OpenShift toolset](https://github.com/openshift/openshift-mcp-server/blob/main/docs/OPENSHIFT.md) — adds one MCP *prompt*, `plan_mustgather`, which generates manifests for a must-gather run (temporary namespace + ServiceAccount + a **cluster-admin** ClusterRoleBinding + privileged pod). It does not add any new read tools. **Enabled here** (see below) — `read_only = true` and `--disable-destructive` block this repo's `claude-code` ServiceAccount from ever actually applying what the prompt generates, so the prompt can plan a must-gather but this identity can't execute one.

This repo's `config.toml` enables `core`, `config`, and `openshift` (see
[SETUP.md §4](SETUP.md#4-hardened-toml-configuration)). Note that OpenShift-specific
resource types (Routes, Projects, ImageStreams, DeploymentConfigs, ...) don't
need the `openshift` toolset at all — `resources_list`/`resources_get` in
`core` already accept any `apiVersion`/`kind`, including OpenShift's, e.g.
`route.openshift.io/v1 Route` or `project.openshift.io/v1 Project`.
