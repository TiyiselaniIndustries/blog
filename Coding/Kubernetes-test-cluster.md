# A Disposable Kubernetes Test Cluster on Proxmox: k3s, Argo CD and a Secrets GUI

I run a production Kubernetes cluster at home, and I wanted somewhere to try apps and manifests without risking it. This post walks through the setup I ended up with: a separate single-node **k3s** VM on Proxmox, **Argo CD** for GitOps deploys, and **OpenBao** with **External Secrets Operator** so I can manage secrets from a web UI.

It includes the problems I actually ran into, because that is where most of the time went.

## What you end up with

- A throwaway k3s VM you can snapshot, break and roll back in seconds
- Argo CD at `http://argocd.test.lan`
- OpenBao (the open-source Vault fork) with a web UI for creating and editing secrets
- External Secrets Operator (ESO) syncing those secrets into normal Kubernetes `Secret` objects

## Why a separate cluster, not a namespace on prod

It is tempting to deploy test apps into a spare namespace on your production cluster. Don't. Namespaces isolate names, not resources. A test app without limits can starve prod pods on the same node, and a bad CRD, privileged pod or broken admission webhook can affect the whole cluster.

A separate VM costs a few GiB of RAM and gives you a safe place to experiment. Because it is a VM, Proxmox snapshots make it truly disposable. Snapshots and rollbacks on a production node are a very different (and risky) story.

## Prerequisites

- A Proxmox host with a few GiB of RAM free. Argo CD idles at roughly 1 GiB, so I used **4 vCPU / 8 GiB** for the VM.
- A cloud-init Ubuntu VM. I created mine from the [community-scripts Ubuntu VM script](https://community-scripts.org/categories?category=operating-systems-and-appliances&preview=ubuntu-vm), which sets up a cloud-init based VM you can adjust afterwards.
- A laptop with SSH and a way to resolve hostnames (a hosts file entry is enough).

Throughout this guide, replace:

| Placeholder | Meaning |
|---|---|
| `192.168.1.9` | The test VM's IP |
| `<vmid>` | The Proxmox VM ID |
| `*.test.lan` | Hostnames for your apps |

## 1. Create the VM

Create the VM from your template or script, then give it a name (I used `testkube`), a static IP or DHCP reservation, and your SSH key.

### Static IP and SSH key via cloud-init

On the Proxmox host:

```bash
qm set <vmid> --ciuser morris --sshkeys ~/laptop.pub
qm set <vmid> --ipconfig0 ip=192.168.1.9/24,gw=192.168.1.1
qm cloudinit update <vmid>
```

Cloud-init applies these on first boot of an instance. For a VM that is already running, add your key to `~/.ssh/authorized_keys` by hand instead:

```bash
mkdir -p ~/.ssh && chmod 700 ~/.ssh
echo 'ssh-ed25519 AAAA... laptop' >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

Install the guest agent so Proxmox can show the VM's IP, and tick **QEMU Guest Agent** under the VM's Options:

```bash
sudo apt install qemu-guest-agent
sudo systemctl enable --now qemu-guest-agent
```

### Gotcha: "REMOTE HOST IDENTIFICATION HAS CHANGED"

If you reuse an IP, your laptop will refuse to connect because the stored host key no longer matches. Confirm it is your VM by comparing fingerprints. On the VM:

```bash
ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
```

If it matches the fingerprint in the warning, remove the stale entry on your laptop and reconnect:

```powershell
ssh-keygen -R 192.168.1.9
ssh morris@192.168.1.9
```

A static IP plus a DHCP reservation stops this from happening on every rebuild. Do not disable strict host key checking globally, since that removes the protection for your other hosts too.

## 2. Install k3s

On the VM:

```bash
curl -sfL https://get.k3s.io | sh -
sudo kubectl get nodes
```

k3s ships with Traefik (ingress), a default storage class and a service load balancer, so most apps work immediately.

### Gotcha: `permission denied` on `/etc/rancher/k3s/k3s.yaml`

The kubeconfig is root-only by default. Give your user a copy rather than loosening permissions on the original, since it contains admin credentials:

```bash
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $USER:$USER ~/.kube/config
chmod 600 ~/.kube/config
```

On k3s, `kubectl` is a symlink to the k3s binary, and it keeps falling back to `/etc/rancher/k3s/k3s.yaml` unless told otherwise. Set the variable so it is used in every shell:

```bash
echo 'export KUBECONFIG=$HOME/.kube/config' >> ~/.bashrc
source ~/.bashrc
kubectl get nodes
```

### Use the cluster from your laptop

```bash
scp morris@192.168.1.9:~/.kube/config ~/.kube/test.yaml
sed -i 's#https://127.0.0.1:6443#https://192.168.1.9:6443#' ~/.kube/test.yaml
export KUBECONFIG=~/.kube/test.yaml
kubectl config rename-context default test
```

Renaming the context means you never confuse the test cluster with prod.

### Snapshot the clean state

On the Proxmox host:

```bash
qm snapshot <vmid> k3s-clean
```

## 3. Install Helm

```bash
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
```

## 4. Install Argo CD

Pick a release tag from the [Argo CD releases page](https://github.com/argoproj/argo-cd/releases) and pin it, so a rebuild gives you the same version:

```bash
export ARGOCD_VERSION=vX.Y.Z

kubectl create ns argocd
kubectl apply -n argocd --server-side \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/$ARGOCD_VERSION/manifests/install.yaml
kubectl -n argocd rollout status deploy/argocd-server
```

Let Argo serve plain HTTP behind Traefik:

```bash
kubectl -n argocd patch cm argocd-cmd-params-cm --type merge \
  -p '{"data":{"server.insecure":"true"}}'
kubectl -n argocd rollout restart deploy argocd-server
```

Create the ingress:

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd
  namespace: argocd
spec:
  ingressClassName: traefik
  rules:
    - host: argocd.test.lan
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: argocd-server
                port:
                  number: 80
EOF
```

### Name resolution

On Windows, edit `C:\Windows\System32\drivers\etc\hosts` as Administrator and add:

```
192.168.1.9  argocd.test.lan
```

Later you can replace this with a wildcard `*.test.lan` record in your router or Pi-hole, since the hosts file needs one line per app.

### Gotcha: 404 when browsing to the IP

If you open `http://192.168.1.9` you get a 404 page. That is Traefik answering: the ingress rule matches only the host `argocd.test.lan`, so a request by IP matches nothing. A 404 here means the ingress controller is working. Use the hostname instead, or test without a hosts entry:

```powershell
curl.exe -H "Host: argocd.test.lan" http://192.168.1.9
```

As a fallback that needs no DNS or ingress:

```bash
kubectl port-forward --address 0.0.0.0 svc/argocd-server -n argocd 8080:80
```

### Log in

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d; echo
```

Log in as `admin`, change the password under User Info, then delete the bootstrap secret:

```bash
kubectl -n argocd delete secret argocd-initial-admin-secret
```

## 5. Choosing a secrets approach (and why I picked a GUI one)

I wanted to create and edit secrets from a web UI. That ruled out my first idea, **Sealed Secrets**, which is a controller plus a CLI with no GUI: its values only exist as encrypted files in Git.

| Approach | GUI | Notes |
|---|---|---|
| OpenBao (or Vault) + External Secrets Operator | Built-in web UI | Secrets live in the vault, not in Git. ESO syncs them into normal Secrets |
| Headlamp / Kubernetes Dashboard | Browse and edit native Secrets | Simpler, but a cluster viewer, not a secrets manager |
| Sealed Secrets | None | Great for pure GitOps, but no UI |

I went with **OpenBao + ESO**. It has the same UI and API as Vault, and it resembles a real production setup.

## 6. Install OpenBao

```bash
helm repo add openbao https://openbao.github.io/openbao-helm
helm repo update
helm search repo openbao/openbao --versions | head   # pick a version and pin it below

cat > openbao-values.yaml <<'EOF'
ui:
  enabled: true
server:
  standalone:
    enabled: true
  dataStorage:
    size: 1Gi
  ingress:
    enabled: true
    ingressClassName: traefik
    hosts:
      - host: bao.test.lan
        paths: []
EOF

helm install openbao openbao/openbao -n openbao --create-namespace \
  --version <pinned-version> -f openbao-values.yaml
```

Add `192.168.1.9  bao.test.lan` to your hosts file. If the ingress doesn't come up, `kubectl -n openbao port-forward svc/openbao 8200:8200` works as a fallback.

The pod shows `0/1 Running` until it is initialized and unsealed. That is expected.

### Initialize and unseal

```bash
kubectl -n openbao exec openbao-0 -- bao operator init -key-shares=1 -key-threshold=1
kubectl -n openbao exec openbao-0 -- bao operator unseal <unseal-key>
```

**Save the unseal key and root token** in a password manager. They are shown once, and they are the keys to everything in the vault. Never commit them to Git.

Standalone mode re-seals after every pod or VM restart, so you will need to run the unseal command again after a reboot. The alternative, `server.dev.enabled: true`, auto-unseals but keeps data in memory only, so it loses everything on restart.

### Enable the KV engine and add a test secret

This was the step I tripped on. Without a KV mount, the CLI returns a misleading `403 ... preflight capability check` error even with the root token, because the `secret/` path doesn't exist yet. Check with `bao secrets list`, then:

```bash
kubectl -n openbao exec -it openbao-0 -- sh
bao login                                   # paste the root token
bao secrets enable -path=secret kv-v2
bao kv put -mount=secret hello API_KEY=test-value-123
bao kv get -mount=secret hello
```

You can do the same in the web UI under **Secrets Engines**.

If `bao secrets list` itself returns 403, a stale `BAO_TOKEN` or `VAULT_TOKEN` environment variable may be overriding your login. Run `unset BAO_TOKEN VAULT_TOKEN` and log in again.

## 7. Install External Secrets Operator

```bash
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets external-secrets/external-secrets \
  -n external-secrets --create-namespace --version <pinned-version>
```

## 8. Connect ESO to OpenBao

For a test cluster, token auth is the simplest. Create a token with a read-only policy rather than handing ESO the root token. Inside the OpenBao pod:

```bash
bao login                                   # root token
bao policy write eso-read - <<'EOF'
path "secret/data/*" { capabilities = ["read"] }
path "secret/metadata/*" { capabilities = ["read", "list"] }
EOF
bao token create -policy=eso-read -period=768h   # copy the token
exit
```

Store the token for ESO and create the `ClusterSecretStore`:

```bash
kubectl -n external-secrets create secret generic bao-token --from-literal=token=<that-token>

cat <<'EOF' | kubectl apply -f -
apiVersion: external-secrets.io/v1
kind: ClusterSecretStore
metadata:
  name: openbao
spec:
  provider:
    vault:
      server: http://openbao.openbao.svc:8200
      path: secret
      version: v2
      auth:
        tokenSecretRef:
          name: bao-token
          namespace: external-secrets
          key: token
EOF

kubectl get clustersecretstore openbao   # should show Valid / Ready
```

If your chart serves an older ESO API, use `external-secrets.io/v1beta1` instead of `v1` in both manifests.

## 9. Test the whole chain

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: hello-creds
  namespace: default
spec:
  refreshInterval: 1m
  secretStoreRef:
    kind: ClusterSecretStore
    name: openbao
  target:
    name: hello-creds
  dataFrom:
    - extract:
        key: hello
EOF

kubectl get externalsecret hello-creds
kubectl get secret hello-creds -o jsonpath='{.data.API_KEY}' | base64 -d; echo
```

You want `SecretSynced` / `True` and your value printed. Edit the value in the OpenBao UI and the Kubernetes Secret follows within a minute. In Git you only commit the `ExternalSecret` file, which contains no secret values, so it is safe in your Argo repo. Your apps then reference the resulting Secret as normal.

### Troubleshooting `SecretSyncedError`

Start with the message:

```bash
kubectl describe externalsecret hello-creds | tail -20
kubectl get clustersecretstore openbao
```

| Cause | Check |
|---|---|
| Secret not in OpenBao | KV v2 enabled at `secret`, and a secret named exactly `hello` exists (my case: the KV engine had never been enabled) |
| OpenBao is sealed | `bao status` shows `Sealed: true`; run the unseal command again |
| Bad token | Recreate `bao-token` with no trailing newline or space |
| Wrong service address | `kubectl -n openbao get svc` and compare with the `server:` URL |
| Permission denied | `bao policy read eso-read` and `bao token lookup <token>` |
| API version mismatch | Switch `v1` to `v1beta1` |

After fixing the cause, force a retry:

```bash
kubectl annotate externalsecret hello-creds force-sync=$(date +%s) --overwrite
```

## 10. Next steps

### GitOps with an app-of-apps repo

Create a repo (e.g. `test-cluster-gitops`) with this layout:

```
bootstrap/
  root-app.yaml
apps/
  hello.yaml
manifests/
  hello/
    deployment.yaml
```

`bootstrap/root-app.yaml` is applied once by hand, and everything else is added by committing to Git:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/<you>/test-cluster-gitops.git
    targetRevision: main
    path: apps
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

Each file under `apps/` is another Argo `Application` pointing at Helm, Kustomize or plain manifests. You can also add OpenBao and ESO there as Applications once you have the manual install working. Test self-healing by scaling a deployment by hand: with `selfHeal: true`, Argo reverts it.

### Keep it isolated from prod

- Don't register the prod cluster in this Argo CD, and don't reuse prod's Argo.
- Use a read-only deploy key if the repo is private.
- Use different secrets in test than in prod. Never copy prod credentials into the less-protected environment.

### Snapshot at known-good points

```bash
qm snapshot <vmid> argo-ready
qm snapshot <vmid> secrets-ready
```

Be aware that rolling back an initialized OpenBao to a snapshot taken before initialization means re-initializing it, so save the new keys again.

### Other ideas

- Add cert-manager for TLS on `*.test.lan`
- Run a small registry (or use GHCR with an image pull secret) for your own images
- Add Headlamp if you also want a general cluster GUI

## Summary of what to back up

| Item | Where to keep it |
|---|---|
| OpenBao unseal key and root token | Password manager |
| ESO read token | Password manager (it can be recreated) |
| Argo CD admin password | Password manager |
| Git repo | Your Git host |

The cluster itself is disposable. With the keys saved and the repo in Git, a rebuild is: clone the VM, install k3s, install Argo CD, apply the root app.
