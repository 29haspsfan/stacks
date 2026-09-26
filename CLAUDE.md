# CLAUDE.md

This repository is a fork of [`cnoe-io/stacks`](https://github.com/cnoe-io/stacks). It holds two different things, and
knowing which one you are touching is the whole point of this file.

| Where                    | What                                                                           | Read by                            |
| ------------------------ | ------------------------------------------------------------------------------ | ---------------------------------- |
| every upstream folder    | upstream's idpbuilder examples (`ref-implementation/`, ...)                    | idpbuilder only                    |
| `homelab/` (to be added) | the lab's packages: AWX, Airflow, dbt, Redis, Superset, local-path-provisioner | Argo CD on `k8s01`, and idpbuilder |

## The lab's packages live in `homelab/`

Argo CD on the lab cluster (`k8s01`, RKE2) has one root Application, named `stacks`. It reads **only the top-level
`*.yaml` files in `homelab/`**, not recursively, and each file is one Argo `Application`.

- **Never use a `cnoe://` source URL in `homelab/`.** Only idpbuilder rewrites `cnoe://`, so plain Argo CD cannot
  resolve it. Use a real URL: this repository for the package's own manifests, the upstream Helm repository for a
  chart. idpbuilder accepts real URLs too, so a file can be tried locally before `k8s01` picks it up.
- **Upstream's `cnoe://` examples are correct as they are.** They are idpbuilder's, not the lab's. Do not "fix" them,
  and do not point the lab at them.
- **Do not ask where the lab's plain Argo CD manifests go.** The answer is `homelab/`, decided on 2026-09-26.

## The contract is written in homelab, not here

The homelab repository (`ekoios/homelab`, cloned at `~/repos/homelab`) owns RKE2, the Argo CD bootstrap, the root
Application, every Namespace and every Secret. What each package must be, and what homelab guarantees in return, is
fixed in two specs there. Read them before adding or changing anything under `homelab/`:

| Spec                                                     | Covers                                                                   |
| -------------------------------------------------------- | ------------------------------------------------------------------------ |
| `superpowers/specs/2026-09-26-k8s01-design.md` section 5 | every package: namespace, pinned source, the Secrets it needs, the rules |
| `superpowers/specs/2026-09-26-awx-design.md` section 6   | AWX's configuration Job and the execution environment built here         |
| `superpowers/my-idp-docs/2026-09-24-canonical-layout.md` | which repository owns what, and where development happens                |

Three of the rules there cause the most damage when missed:

- **No credential, Secret or Namespace in any file.** homelab creates them all before the root Application syncs, and a
  package only refers to a Secret by name.
- **Adopt, reconcile, then automate.** The first commit of a package has no `syncPolicy.automated`; automation is a
  separate commit once `argocd app diff` is empty.
- **No `Certificate` and no `secretName`.** Every Ingress uses the cluster's `default` TLSStore.

If a question is not answered by those specs, it is a question for the homelab specs, not a local decision here.

## Where this runs

Development happens on `desktop01`, not the Mac. idpbuilder runs there, in its own `kind` cluster, to try a package
before Argo CD on `k8s01` syncs it.

## Fork discipline

`LOCAL-CHANGES.md` lists every change that exists here and not upstream. Add a row there in the same PR as any such
change, because each one is a conflict waiting to happen on the next rebase onto `upstream/main`.
