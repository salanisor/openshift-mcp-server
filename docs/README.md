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

- [Tools and Functionalities](https://github.com/openshift/openshift-mcp-server#tools-and-functionalities) — the full list of every MCP tool this fork exposes, grouped by toolset. **Read the note below first** — this repo does not run the plain community build, so the toolset list here (`cluster-diagnostics`, `cni-diagnostics`, `config`, `core`, `helm`, `kcp`, `kubevirt`, `netedge`, `netobserv`, `oadp`, `observability/*`, `openshift`, `openshift/mustgather`, `ossm`, `ovn-kubernetes`, `tekton`) is the real one that applies here — bigger than what `npx kubernetes-mcp-server@latest` alone would give you.
- [Configuration reference](https://github.com/openshift/openshift-mcp-server/blob/main/docs/configuration.md) — every `config.toml` option and CLI flag (`--read-only`, `--disable-destructive`, `--toolsets`, `--config`, ...).
- [Getting started with Claude Code](https://github.com/openshift/openshift-mcp-server/blob/main/docs/getting-started-claude-code.md) — upstream's own Claude Code integration guide.
- [KubeVirt toolset](https://github.com/openshift/openshift-mcp-server/blob/main/docs/kubevirt.md) — adds the read-only `vm_guest_info` tool plus the `vm-troubleshoot`/`windows-golden-image` MCP prompts (OpenShift Virtualization-branded on the image this repo runs). Also defines write tools (`vm_create`, `vm_lifecycle`, `vm_clone`) that stay hidden while `read_only = true` (confirmed live via `tools/list`, 2026-08-06). **Enabled here.**
- [OpenShift toolset](https://github.com/openshift/openshift-mcp-server/blob/main/docs/OPENSHIFT.md) — adds the `plan_mustgather` prompt only, no new tools. **Enabled here.**

> **Important (confirmed 2026-08-06): `openshift/openshift-mcp-server` is a
> downstream Red Hat fork of the community `containers/kubernetes-mcp-server`
> project, and this repo runs the fork, not the community build.** The
> fork's extra toolsets (`openshift`, `openshift/mustgather`, `oadp`,
> `netedge`, `cluster-diagnostics`, `cni-diagnostics`, `ovn-kubernetes`) only
> exist in Red Hat's **Tech Preview container image**
> (`registry.redhat.io/openshift-mcp-tech-preview/openshift-mcp-server-rhel9`)
> — there's no npm package for the fork. `npx kubernetes-mcp-server@latest`
> (this repo's setup until 2026-08-06) installs the plain community package
> and never had `openshift` at any point. See
> [SETUP.md §3](SETUP.md#3-install-and-configure-the-mcp-server) for the
> pull-secret + podman flow this repo now uses, and §4 for the known
> tool-availability gap between the two builds (this image is missing
> `configuration_view` and the heuristic `vm_troubleshoot` tool that the
> community build has).

This repo's `config.toml` enables `core`, `config`, `kubevirt`, and
`openshift` (see [SETUP.md §4](SETUP.md#4-hardened-toml-configuration)).
Note that OpenShift-specific resource types (Routes, Projects, ImageStreams,
DeploymentConfigs, ...) don't need any extra toolset for basic reads —
`resources_list`/`resources_get` in `core` already accept any
`apiVersion`/`kind`, including OpenShift's, e.g. `route.openshift.io/v1
Route` or `project.openshift.io/v1 Project`.
