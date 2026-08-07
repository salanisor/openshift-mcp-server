# openshift-mcp-server — Secure Setup Guide

Source project: https://github.com/openshift/openshift-mcp-server

This folder is the single source of truth for how this MCP server is configured
in this environment. Update it whenever the setup changes — don't let the
real config drift from what's documented here. Identity names below
(`cluster-ops` namespace, `claude-code` ServiceAccount) match what's actually
applied in this repo's `templates/namespace.yaml` / `templates/serviceaccount.yaml` /
`templates/clusterrolebinding.yaml` (or `templates/rolebinding.yaml` for
Option A) — if you rename the resources, update this doc in the same
change.

Threat model in one line: this server gives an LLM a live credential into a
Kubernetes/OpenShift cluster. Every choice below defaults to the option that
limits what a compromised or misbehaving model session can do.

---

## 1. Core security principles (apply all of these)

1. **Never point it at your personal kubeconfig / admin credentials.**
   Create a dedicated, least-privilege ServiceAccount (§2).
2. **Run `--read-only` unless write access is a hard requirement.**
3. **Explicitly deny `Secret` (and any other sensitive kind) via `denied_resources`**,
   even in read-only mode — read-only still allows *reading* secret values.
4. **Prefer short-lived tokens over static ones.** `oc create token` tokens
   expire (default 1h, set explicitly below); rotate rather than mint
   long-lived service account tokens.
5. **Scope to a namespace with a `Role`/`RoleBinding` instead of a
   `ClusterRole`/`ClusterRoleBinding`** wherever the work doesn't need
   cluster-wide visibility. Verify the grant actually landed narrow — don't
   just trust the manifest (see "Verify" under §2).
6. **Only enable the toolsets you need** (`--toolsets`) — each toolset is
   attack surface. Default is `core,config`; don't add `helm`, `kubevirt`,
   etc. unless you're actually using them (see §4 for what each covers).
7. **Keep `--disable-destructive` on** for any session that shouldn't delete
   or mutate resources, independent of read-only.
8. **Restrict multi-cluster access** (`--disable-multi-cluster`) if you only
   ever need one cluster context — reduces blast radius if the model picks
   the wrong context.
9. **File permissions**: kubeconfig files and any file holding a bare token
   must be `chmod 600`. This includes ad hoc/scratch token files, not just
   the final kubeconfig — see §2.
10. **Stay current** — only the latest release gets security fixes (see §6).
11. **Have a revocation path ready** before you need it (see §8) — know how
    to kill the token and the ServiceAccount's access without waiting on a
    natural expiry.

---

## 2. Create a dedicated, least-privilege ServiceAccount

Do this instead of using your own kubeconfig.

```bash
# Dedicated namespace + identity for the MCP server
oc create namespace cluster-ops
oc create serviceaccount claude-code -n cluster-ops
```

**Declarative alternative:** `templates/namespace.yaml` and
`templates/serviceaccount.yaml` in this directory are the YAML equivalents
of the two commands above — apply them with:

```bash
oc apply -f templates/namespace.yaml -f templates/serviceaccount.yaml
```

Keep the manifests and the imperative commands in sync — don't let one drift
from the other.

### Grant read-only permissions — prefer namespace-scoped

**Option A — namespace-scoped (preferred, smallest blast radius):**

```bash
oc create rolebinding claude-code-rb \
    --role=view \
    --serviceaccount=cluster-ops:claude-code \
    -n <target-namespace>
```

**Option B — cluster-wide (only if you genuinely need visibility across
all namespaces; understand this lets the model read cluster-wide state):**

```bash
oc create clusterrolebinding claude-code-crb \
    --clusterrole=view \
    --serviceaccount=cluster-ops:claude-code
```

The built-in `view` ClusterRole already excludes `Secret` contents in most
distros, but don't rely on that alone — also set `denied_resources` in the
TOML config (§4).

The binding template in `templates/` should match whichever option you
picked: `templates/rolebinding.yaml`, a `RoleBinding` (namespaced, carries
its own `metadata.namespace`), for Option A, or
`templates/clusterrolebinding.yaml`, a `ClusterRoleBinding` (cluster-scoped
— any `metadata.namespace` on it is meaningless and should be omitted), for
Option B. This repo currently has Option B applied. Apply it the same way:

```bash
oc apply -f templates/clusterrolebinding.yaml
```

**Verify the grant is as narrow as intended** — don't just trust the
manifest:

```bash
oc auth can-i get secrets \
    --as=system:serviceaccount:cluster-ops:claude-code -n cluster-ops  # expect: no
oc auth can-i get pods \
    --as=system:serviceaccount:cluster-ops:claude-code -n cluster-ops  # expect: yes
oc auth can-i get pods \
    --as=system:serviceaccount:cluster-ops:claude-code --all-namespaces        # expect: no, unless Option B was chosen deliberately
```

### Generate a short-lived token (don't skip `--duration`)

```bash
TOKEN="$(oc create token claude-code --duration=1h -n cluster-ops)"
```

If you save `$TOKEN` to a standalone file for any reason (not just embedded
in the kubeconfig below), `chmod 600` that file immediately and delete it
once it's no longer needed — a bare token file is as sensitive as the
kubeconfig and shouldn't outlive the token's `--duration`.

### Build an isolated kubeconfig — never merge into your default one

```bash
API_SERVER="$(oc config view --minify -o jsonpath='{.clusters[0].cluster.server}')"
KUBECONFIG_FILE="$HOME/claude/openshift-mcp-server.kubeconfig"

# `oc login` kubeconfigs normally embed the CA as base64 data rather than a
# file path, so certificate-authority is empty — extract it from the raw
# (unredacted) view instead. --raw is required or the data field is
# replaced with "DATA+OMITTED".
CA_FILE="$(mktemp)"
oc config view --minify --raw -o jsonpath='{.clusters[0].cluster.certificate-authority-data}' \
    | base64 -d > "$CA_FILE"

oc config --kubeconfig="$KUBECONFIG_FILE" set-cluster claude-code-cluster \
    --server="$API_SERVER" \
    --certificate-authority="$CA_FILE" \
    --embed-certs=true

rm -f "$CA_FILE"   # cert is now embedded in $KUBECONFIG_FILE, temp file no longer needed

oc config --kubeconfig="$KUBECONFIG_FILE" set-credentials claude-code \
    --token="$TOKEN"

oc config --kubeconfig="$KUBECONFIG_FILE" set-context claude-code-context \
    --cluster=claude-code-cluster \
    --user=claude-code

oc config --kubeconfig="$KUBECONFIG_FILE" use-context claude-code-context

# Required — this file holds a live bearer token
chmod 600 "$KUBECONFIG_FILE"
```

Never use `--insecure-skip-tls-verify` as a shortcut here — it disables
certificate validation entirely and exposes the connection to
man-in-the-middle interception. Extracting the real CA as shown above avoids
needing it.

**Sanity-check the kubeconfig actually has a credential** — an interrupted
`set-credentials` step leaves `users:` empty (`user: {}`) with no error:

```bash
oc whoami --kubeconfig="$KUBECONFIG_FILE"
oc get ns --kubeconfig="$KUBECONFIG_FILE" --request-timeout=5s   # should list only what Option A/B granted
```

### Token renewal (tokens expire — this is intentional)

```bash
TOKEN="$(oc create token claude-code --duration=2h -n cluster-ops)"
oc config --kubeconfig="$KUBECONFIG_FILE" set-credentials claude-code --token="$TOKEN"
```

Automate renewal (cron/systemd timer) rather than reaching for a
non-expiring token out of convenience.

---

## 3. Connect to Claude Code

### Prerequisite: install the Claude Code CLI

Check first — don't assume it's there:

```bash
claude --version   # should print e.g. "2.1.211 (Claude Code)"
```

If that fails with `command not found`, install it. On Fedora, the
package-manager route is the most maintainable (integrates with normal
`dnf` updates and GPG-verifies the repo):

```bash
sudo tee /etc/yum.repos.d/claude-code.repo <<'EOF'
[claude-code]
name=Claude Code
baseurl=https://downloads.claude.ai/claude-code/rpm/stable
enabled=1
gpgcheck=1
gpgkey=https://downloads.claude.ai/keys/claude-code.asc
EOF
sudo dnf install claude-code
```

`dnf` will prompt you to confirm the signing key's fingerprint before
trusting it — it must read
`31DD DE24 DDFA B679 F42D 7BD2 BAA9 29FF 1A7E CACE`. Don't accept a
mismatch. This installs the `stable` channel; updates then come through
`sudo dnf upgrade claude-code`, not Claude Code's own auto-updater.

**Alternative — npm global install** (this machine already has Node.js 22+
and npm with `~/.npm-global/bin` on `PATH`, so this needs no `sudo` and no
extra PATH setup):

```bash
npm install -g @anthropic-ai/claude-code
```

Upgrade later with `npm install -g @anthropic-ai/claude-code@latest` (not
`npm update -g`, which won't necessarily move you to the latest version).

Either way, the MCP server itself also runs through `npx` (see the
`add-json` command below), so keep Node/npm current regardless of which
install method you pick for the `claude` CLI.

### Quick setup (recommended — hardened config from §4 baked in)

This is the actual command used in this environment — points at the
hardened `config.toml` (§4) rather than just `--read-only`, and layers on
`--disable-destructive` / `--disable-multi-cluster` per §1:

```bash
claude mcp add-json kubernetes-mcp-server \
  '{"command":"npx","args":["-y","kubernetes-mcp-server@latest","--config","'${HOME}'/claude/openshift-mcp-server/config.toml","--disable-destructive","--disable-multi-cluster"],"env":{"KUBECONFIG":"'${HOME}'/claude/openshift-mcp-server.kubeconfig"}}' \
  -s user
```

If you haven't written `config.toml` yet, do §4 first, or fall back to
bare `--read-only` temporarily — but replace it once §4 is done.

Note this points `KUBECONFIG` at the dedicated kubeconfig built in §2 —
**not** your personal `~/.kube/config`.

**`add-json` is not idempotent — it silently no-ops if the name is already
registered.** `claude mcp add-json` has no `--force` or update option (check
`claude mcp add-json --help`). If `kubernetes-mcp-server` already exists
under this name, re-running the command above prints
`MCP server kubernetes-mcp-server already exists in user config` and makes
**no change** — not to the command, not to `args`, not to `env`. This has
bitten this setup more than once: editing `config.toml`'s *contents*
(toolsets, `denied_resources`, `read_only`) is safe to do in place, because
`npx` re-reads that file on every process spawn. But changing the
*invocation itself* — the `--config` path, `--disable-destructive`,
`--disable-multi-cluster`, or the `KUBECONFIG` env var — requires replacing
the registration, not just re-running `add-json`:

```bash
# 1. Remove the stale registration
claude mcp remove kubernetes-mcp-server -s user

# 2. Re-add with the current, intended invocation
claude mcp add-json kubernetes-mcp-server \
  '{"command":"npx","args":["-y","kubernetes-mcp-server@latest","--config","'${HOME}'/claude/openshift-mcp-server/config.toml","--disable-destructive","--disable-multi-cluster"],"env":{"KUBECONFIG":"'${HOME}'/claude/openshift-mcp-server.kubeconfig"}}' \
  -s user

# 3. Verify the new invocation actually landed (add-json prints no confirmation
#    of what it registered, so check the raw config directly)
grep -A 12 '"kubernetes-mcp-server"' ~/.claude.json
```

**Whenever you change anything about *how* the server is invoked** — the
`--config` path, any CLI flag, `KUBECONFIG`, or the command itself — treat
`remove` + re-add as a required step, not an optional one. `claude mcp list`
showing `✔ Connected` afterward is not proof the new invocation took: it
only reports the existing connection's health, so always confirm against
`~/.claude.json` (step 3 above) after any invocation change.

### Manual config alternative

File: `~/.config/claude-code/config.toml`

```toml
[[mcp_servers]]
name = "kubernetes-mcp-server"
command = "npx"
args = ["-y", "kubernetes-mcp-server@latest", "--config", "/home/YOUR_USERNAME/claude/openshift-mcp-server/config.toml", "--disable-destructive", "--disable-multi-cluster"]

[mcp_servers.env]
KUBECONFIG = "/home/YOUR_USERNAME/claude/openshift-mcp-server.kubeconfig"
```

Replace `/home/YOUR_USERNAME/` with your actual home directory.

### Verify

```bash
claude mcp list
```
Confirms the server shows a healthy/connected status.

---

## 4. Hardened TOML configuration

Prefer a config file over passing everything as CLI flags — it's easier to
audit and diff. Point the server at it with `--config`. `config.toml` in
this directory is exactly the template below — edit it in place rather than
letting a copy drift, and re-run §3's quick setup (or restart the MCP
connection) after changing it.

```toml
log_level = 2
read_only = true
toolsets = ["core", "config"]   # only what you actually use

# Always deny Secrets even though read_only is set —
# read-only still permits *reading* secret values otherwise.
[[denied_resources]]
group = ""
version = "v1"
kind = "Secret"

# Add other sensitive kinds as needed, e.g. cluster-scoped RBAC objects
# [[denied_resources]]
# group = "rbac.authorization.k8s.io"
# kind = "ClusterRoleBinding"
```

Key flags to know (combine as needed):

| Flag | Effect |
|---|---|
| `--read-only` | Blocks all write operations |
| `--disable-destructive` | Blocks delete/update specifically (independent of read-only) |
| `--disable-multi-cluster` | Locks to a single cluster context |
| `--toolsets` | Comma-separated allowlist of enabled tool groups |
| `--config` | Path to the TOML file above |
| `--kubeconfig` | Path to the dedicated kubeconfig from §2 |

Toolsets and what they add (attack surface grows with each one enabled):

| Toolset | Adds |
|---|---|
| `core` | Basic resource read/list/describe — the baseline, always needed |
| `config` | Reading cluster/kubeconfig context info |
| `helm` | Helm release install/list/uninstall — write-capable even under some read-only interpretations; only enable if Helm workflows are actually in use |
| `kubevirt` | VirtualMachine start/stop/access — treat as equivalent to node-level access on any cluster running KubeVirt |

Sensitive data (tokens, keys, passwords, cloud credentials) is automatically
redacted in MCP logging output — this is a defense-in-depth measure, not a
substitute for `denied_resources` and least-privilege RBAC.

---

## 5. Optional: enterprise SSO instead of static tokens

If available in your environment, prefer OIDC-backed auth over long-lived
ServiceAccount tokens. These docs live in the upstream project, not this
repo — see:

- Keycloak OIDC: upstream `docs/KEYCLOAK_OIDC_SETUP.md`
- Microsoft Entra ID: upstream `docs/ENTRA_ID_SETUP.md`

(https://github.com/openshift/openshift-mcp-server/tree/main/docs)

These give you centrally revocable, short-lived credentials instead of a
token you have to manage and rotate by hand.

---

## 6. Vulnerability handling

- **Only the latest release receives security fixes** — no LTS/backport
  branches. Pin to `@latest` deliberately, and check for updates regularly.
- **Report vulnerabilities privately**, not via public issues: use the
  repo's "Report a vulnerability" page under the Security tab (GitHub
  private advisory form). Include issue type, affected version, repro
  steps, PoC, and impact.
- Downstream/Red Hat product issues are handled through Red Hat's internal
  product security process in addition to upstream.

---

## 7. Toolset and TOML reference

See §4 for the full toolset table and the hardened config template.

---

## 8. If a token or kubeconfig leaks

Don't wait for natural expiry — revoke immediately:

```bash
# Delete the ServiceAccount's active token secrets/bindings by rotating identity:
# simplest reliable option is to delete and recreate the ServiceAccount, which
# invalidates every token issued for the old UID.
oc delete serviceaccount claude-code -n cluster-ops
oc create serviceaccount claude-code -n cluster-ops
oc apply -f templates/clusterrolebinding.yaml   # rebind the new SA's UID to the view role

# Then mint a fresh token and rebuild the kubeconfig per §2.
```

Also delete any leaked standalone token file and the old kubeconfig, and
audit `oc get events -n <target-namespace>` / cluster audit logs for the
window the credential was live, if your cluster retains them.

---

## 9. Pre-flight checklist

Before pointing this server at any real cluster, confirm:

- [ ] Using dedicated `claude-code` ServiceAccount, not personal/admin creds
- [ ] RBAC binding is namespace-scoped (`RoleBinding`) unless cluster-wide is
      truly required (`ClusterRoleBinding`) — and the matching file in
      `templates/` (`rolebinding.yaml` or `clusterrolebinding.yaml`) reflects
      whichever was chosen
- [ ] `oc auth can-i get secrets --as=system:serviceaccount:cluster-ops:claude-code` returns `no`
- [ ] Token has an explicit, short `--duration` and a renewal plan
- [ ] `read_only = true` unless writes are explicitly required for the task
- [ ] `denied_resources` blocks `Secret` at minimum
- [ ] `--disable-destructive` set if writes are enabled but deletes aren't needed
- [ ] `toolsets` trimmed to only what's used
- [ ] kubeconfig and any bare token files are `chmod 600`
- [ ] `oc whoami --kubeconfig=...` succeeds (kubeconfig actually has a credential, not an empty `user: {}`)
- [ ] Running the latest released version
- [ ] `KUBECONFIG` env var in the Claude MCP config points at the isolated
      file (`~/claude/openshift-mcp-server.kubeconfig`), not the default
      `~/.kube/config`
- [ ] Revocation steps (§8) are known, not something to figure out during an incident
