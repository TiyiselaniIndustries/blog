---
title: 6. GitOps and Argo CD
description: CI never touches the cluster. It commits a chart; Argo CD notices and applies it.
series: Deployment blueprint
seriesOrder: 7
publishedAt: 2026-11-25T08:06:00Z
draft: false
---

CI never touches the cluster. It commits a chart; Argo CD notices and applies it.

## Repo layout

`TiyiselaniIndustries/tmlabs-gitops`:

```
apps/
  tmlabs.yaml                 Argo Application → manifests/tmlabs
  tmlabs-extras.yaml          Argo Application → manifests/tmlabs-extras
manifests/
  tmlabs/                     CI-owned; replaced every deploy
  tmlabs-extras/              hand-maintained: Ingress + ExternalSecrets
README.md
```

`manifests/tmlabs` is owned by CI, `manifests/tmlabs-extras` by you. Only the
second is safe to edit.

## The two Applications

| | `tmlabs` | `tmlabs-extras` |
| --- | --- | --- |
| Path | `manifests/tmlabs` | `manifests/tmlabs-extras` |
| Source type | **Helm** | **Directory** |
| Namespace | `tmlabs` | `tmlabs` |
| Contents | generated chart | Ingress, ExternalSecrets |
| Created by | CI (chart), you (Application) | you |

Both carry:

```yaml
syncPolicy:
  automated: { prune: true, selfHeal: true }
  syncOptions: [CreateNamespace=true]
```

- **prune** — deletes resources whose manifest disappeared from Git.
- **selfHeal** — reverts manual `kubectl edit`s. Don't debug by editing live
  objects; you'll be reverted. Change the manifest.
- **CreateNamespace** — creates `tmlabs` before the first sync.

## Bootstrap

### Repo credential

Argo needs read access to the private repo. A fine-grained PAT with
*Contents: read*, stored as a labelled Secret in the `argocd` namespace:

```bash
kubectl -n argocd create secret generic tmlabs-gitops-repo \
  --from-literal=type=git \
  --from-literal=url=https://github.com/TiyiselaniIndustries/tmlabs-gitops.git \
  --from-literal=username=x-access-token \
  --from-literal=password=<PAT>

kubectl -n argocd label secret tmlabs-gitops-repo \
  argocd.argoproj.io/secret-type=repository
```

(The UI equivalent is *Settings → Repositories*.)

### Applications

```bash
kubectl apply -f apps/
kubectl -n argocd get applications
```

Or through the UI — *Applications → + NEW APP*, filling the same fields. Don't do
both; the second apply fights the first over the same object names.

**Sync `tmlabs-extras` first.** The ExternalSecrets must exist before the app's
pods start, or they come up without configuration.

## The push step

```bash
CHECKOUT="$RUNNER_TEMP/gitops-${GITHUB_RUN_ID}-${GITHUB_RUN_ATTEMPT}"
rm -rf "$CHECKOUT"
git clone "https://x-access-token:${{ secrets.GITOPS_TOKEN }}@github.com/${GITOPS_REPO}.git" "$CHECKOUT"
rm -rf "$CHECKOUT/manifests/tmlabs"
mkdir -p "$CHECKOUT/manifests/tmlabs"
cp -r chart/. "$CHECKOUT/manifests/tmlabs/"
cd "$CHECKOUT"
git add -A
if git diff --cached --quiet; then
  echo "No chart changes to publish."
  exit 0
fi
git -c user.name=ci -c user.email=ci@local commit -m "deploy ${TAG}"
git pull --rebase
git push
```

Design points worth keeping when you copy this:

- **unique temp path** — self-hosted runners reuse their workspace.
- **`git diff --cached --quiet` guard** — `commit || exit 0` silently skips the
  push and reports success. Exit only when there is genuinely no diff.
- **`pull --rebase` before `push`** — the commit is created first, so a
  concurrent change is integrated rather than rejected.
- **replacing the whole directory** (`rm -rf` then copy) guarantees deleted
  templates actually disappear; a plain copy would leave them behind and Argo's
  prune would never fire.

## Trap: "path does not contain a chart"

Expected before the first CI push, because `manifests/tmlabs` contains only
`.gitkeep`. It clears when CI commits a real `Chart.yaml`. Create the `tmlabs`
Application anyway — nothing is broken.

## Trap: ReplicaSets accumulate

After a few deploys the resource tree shows `rev:1`, `rev:2`, … Those are
**rollback history**, not leaks. Each Deployment sets `revisionHistoryLimit: 3`,
so Kubernetes keeps the current ReplicaSet plus three old ones and deletes the
oldest on each rollout. Argo doesn't manage them; the Deployment controller does.
To keep fewer, post-process in Finalize:

```bash
for f in chart/templates/*/deployment.yaml; do
  yq -i '.spec.revisionHistoryLimit = 1' "$f"
done
```

Next: [07-configuration-and-secrets.md](https://github.com/TiyiselaniIndustries/tmlabs-backend/blob/main/docs/blueprint/07-configuration-and-secrets.md)
