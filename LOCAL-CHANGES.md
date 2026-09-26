# Local changes in this fork

This repository is a fork of [`cnoe-io/stacks`](https://github.com/cnoe-io/stacks). Everything below is a change that
exists **here and not upstream**. Keep this file current when you rebase onto upstream, because each entry is a conflict
waiting to happen.

| Date       | Area                          | Change                                                 | Upstream status  |
| ---------- | ----------------------------- | ------------------------------------------------------ | ---------------- |
| 2026-08-25 | `ref-implementation/keycloak` | Download `kubectl` for the node's own CPU architecture | Not reported yet |
| 2026-09-26 | `CLAUDE.md`                   | Agent guide: the lab's packages go in `homelab/`       | Fork-only        |

## 2026-08-25 - Keycloak config job downloaded an amd64 `kubectl` on arm64 nodes

**File:** `ref-implementation/keycloak/manifests/keycloak-config.yaml` (the `config` Job)

### Symptom

On an Apple Silicon (arm64) machine, `backstage` and `argo-workflows` never leave `OutOfSync` / `Degraded`, and the
`backstage` namespace has no pods at all. ArgoCD reports:

```shell
$ one or more synchronization tasks completed unsuccessfully due to application controller sync timeout
```

The Backstage `ExternalSecret` explains itself more clearly:

```shell
$ error retrieving secret at .data[0], key: keycloak-clients, err: secrets "keycloak-clients" not found
```

Meanwhile the Keycloak `config` Job reports **success**, which is what makes this confusing.

### Root cause

The Job's script hardcoded an `amd64` download:

```bash
curl -sS -LO "https://dl.k8s.io/release/v1.28.3//bin/linux/amd64/kubectl"
```

On an arm64 node that URL returns a `NoSuchKey` XML error page with HTTP 404, and without `--fail-with-body` `curl`
happily saves the error page as a file named `kubectl`. The script then ran `./kubectl` and got:

```text
./kubectl: line 1: syntax error near unexpected token `<'
```

That happens at the **last** step, after the realm, clients, groups and users are all created, but _before_ the script
writes the `keycloak-clients` secret that Backstage and Argo Workflows read their OIDC credentials from.

A second bug then hid the first. The top of the script had an "have I already run?" guard:

```bash
curl --fail-with-body ... "${KEYCLOAK_URL}/admin/realms/cnoe" &> /dev/null
if [ $? -eq 0 ]; then
  exit 0
fi
```

Because the interrupted run had already created the `cnoe` realm, every retry hit that guard, exited `0`, and the Job
was marked Complete. The secret was therefore never created and never would be.

### Fix

Two changes, both in the `config` Job script:

1. Resolve the architecture at runtime and fail loudly on a bad download.

   ```bash
   KUBECTL_ARCH=$(dpkg --print-architecture)
   curl -sS --fail-with-body -LO "https://dl.k8s.io/release/v1.28.3/bin/linux/${KUBECTL_ARCH}/kubectl"
   chmod +x kubectl
   ./kubectl version --client > /dev/null
   ```

   `dpkg --print-architecture` returns exactly the strings `dl.k8s.io` expects (`amd64`, `arm64`), and the image is
   Ubuntu, so it is always present. The download also moved **above** the guard, because the guard now needs `kubectl`.

2. Only skip when the work is genuinely finished, and self-heal when it is not.

   ```bash
   curl --fail-with-body ... "${KEYCLOAK_URL}/admin/realms/cnoe" &> /dev/null
   REALM_EXISTS=$?
   ./kubectl -n keycloak get secret keycloak-clients &> /dev/null
   CLIENTS_SECRET_EXISTS=$?

   if [ $REALM_EXISTS -eq 0 ] && [ $CLIENTS_SECRET_EXISTS -eq 0 ]; then
     exit 0
   fi

   if [ $REALM_EXISTS -eq 0 ]; then
     # a previous run was interrupted between the realm and the secret
     curl -sS --fail-with-body ... -X DELETE "${KEYCLOAK_URL}/admin/realms/cnoe" &> /dev/null
   fi
   ```

   The `keycloak-config` Role already grants `get` on secrets in the `keycloak` namespace, so no RBAC change was needed.

### Recovering a cluster that already hit this

The fix in this repo does not retroactively repair a stuck cluster, because ArgoCD will not rerun a Job that is marked
Complete against an unchanged manifest. Either recreate the cluster from this fork, or repair in place: delete the
`cnoe` realm through the Keycloak admin API, delete the `config` Job in the `keycloak` namespace, and sync the
`keycloak` ArgoCD application so the Job is recreated and runs fully.

### Verification

- Applied on a live arm64 `idpbuilder` kind cluster: the `config` Job completed in 19 seconds and all ten ArgoCD
  applications reached `Synced` / `Healthy`.
- `https://cnoe.localtest.me:8443/backstage` returned HTTP 200.
- `kubectl create -f ... --dry-run=client` parses the manifest, and `bash -n` on the extracted Job script passes.

## Rebasing onto upstream

```bash
git -C ~/repos/stacks fetch upstream
git -C ~/repos/stacks rebase upstream/main
```

Expect a conflict in `ref-implementation/keycloak/manifests/keycloak-config.yaml` until the change above is accepted
upstream. Re-read this file before resolving it.
