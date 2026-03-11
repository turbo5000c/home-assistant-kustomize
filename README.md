# Home Assistant Kustomize Deployment

This repository contains a Kustomize setup for deploying [Home Assistant](https://www.home-assistant.io/) on Kubernetes. It is designed to be consumed directly by [FluxCD](https://fluxcd.io/) so that the average GitOps user can get Home Assistant running with minimal configuration.

## Repository Contents

| File | Description |
|---|---|
| `kustomization.yaml` | Root Kustomize manifest – lists all resources |
| `namespace.yaml` | Creates the `home-assistant` namespace |
| `statefulset.yaml` | Home Assistant StatefulSet with a built-in PVC |
| `service.yaml` | ClusterIP Service on port 8123 |
| `ingressroute.yaml` | Optional Traefik `IngressRoute` (not included by default) |
| `flux/gotk-sync.yaml` | Ready-to-use FluxCD `GitRepository` + `Kustomization` manifests |

---

## Prerequisites

- A running Kubernetes cluster (tested with k0s, k3s, and microk8s)
- `kubectl` configured to talk to your cluster
- [FluxCD](https://fluxcd.io/flux/installation/) bootstrapped on the cluster *(for GitOps deployment)*
- A default or named [StorageClass](https://kubernetes.io/docs/concepts/storage/storage-classes/) available in your cluster

---

## Option A – Direct kubectl Deployment

### 1. Clone the Repository

```bash
git clone https://github.com/turbo5000c/home-assistant-kustomize.git
cd home-assistant-kustomize
```

### 2. (Optional) Customise the Manifests

| File | Common changes |
|---|---|
| `statefulset.yaml` | Change the image tag, resource requests, or storage size |
| `service.yaml` | Change `type` to `NodePort` or `LoadBalancer` |
| `kustomization.yaml` | Uncomment `ingressroute.yaml` to enable Traefik ingress |
| `ingressroute.yaml` | Set your hostname and TLS secret name |

### 3. Apply with Kustomize

```bash
kubectl apply -k .
```

### 4. Verify the Deployment

```bash
kubectl get all -n home-assistant
```

Home Assistant will be available at `http://<cluster-node-ip>:8123` once the pod is `Running`.

---

## Option B – FluxCD (GitOps) Deployment

This is the recommended approach for users who already have FluxCD running.

### 1. Add the Flux Manifests to Your GitOps Repository

Copy `flux/gotk-sync.yaml` from this repository into your Flux GitOps repo (typically under `clusters/<cluster-name>/` or `apps/`):

```bash
# Inside your Flux GitOps repository
cp /path/to/home-assistant-kustomize/flux/gotk-sync.yaml clusters/my-cluster/home-assistant.yaml
```

Or reference the file directly in your existing Flux `Kustomization`:

```yaml
# In your Flux GitOps repo – e.g. clusters/my-cluster/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - home-assistant.yaml        # the copied gotk-sync.yaml
```

### 2. Understand `flux/gotk-sync.yaml`

The file contains two Flux resources, both deployed into the `flux-system` namespace (where your Flux controllers live):

```yaml
# Tells Flux where to pull the Home Assistant manifests from
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: home-assistant
  namespace: flux-system
spec:
  interval: 2h
  ref:
    semver: ">=2024.1.1"   # tracks the latest semver tag; change to branch: main if preferred
  url: https://github.com/turbo5000c/home-assistant-kustomize.git

---
# Tells Flux which path to reconcile and into which namespace to deploy
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: home-assistant
  namespace: flux-system
spec:
  interval: 4h
  path: "./"
  prune: true
  sourceRef:
    kind: GitRepository
    name: home-assistant
    namespace: flux-system
  targetNamespace: home-assistant
```

> **Why `namespace: flux-system`?**  
> The `GitRepository` and `Kustomization` Flux CRDs must be created in the same namespace as the Flux controllers (default: `flux-system`). The actual Home Assistant pods land in `targetNamespace: home-assistant`.

### 3. Commit and Push

```bash
git add clusters/my-cluster/home-assistant.yaml
git commit -m "feat: add home-assistant deployment"
git push
```

Flux will detect the change within its reconciliation interval and deploy Home Assistant automatically.

### 4. Verify the Deployment

```bash
# Check Flux reconciliation status
flux get kustomizations home-assistant

# Check pod status
kubectl get pods -n home-assistant
```

---

## Storage Configuration

The PVC is defined inside the `StatefulSet` under `volumeClaimTemplates`. By default:

```yaml
volumeClaimTemplates:
  - metadata:
      name: home-assistant
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 5Gi
```

To change the storage class or size, edit `statefulset.yaml` and add/modify the `storageClassName` field:

```yaml
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: longhorn   # or local-path, ceph-rbd, etc.
        resources:
          requests:
            storage: 10Gi
```

### Using HostPath Storage via Flux Patch

If you prefer to bind-mount a directory from the node instead of using a PVC, add the following `patches` block to the `Kustomization` in `flux/gotk-sync.yaml`:

```yaml
  patches:
    - patch: |
        - op: add
          path: /spec/template/spec/volumes/-
          value:
            name: home-assistant-config
            hostPath:
              path: /opt/home-assistant/config
              type: DirectoryOrCreate
        - op: replace
          path: /spec/template/spec/containers/0/volumeMounts/0
          value:
            name: home-assistant-config
            mountPath: /config
      target:
        kind: StatefulSet
        name: home-assistant
```

---

## Ingress / Reverse Proxy

### Traefik IngressRoute

Uncomment `ingressroute.yaml` in `kustomization.yaml` and update the hostname and TLS secret:

```yaml
# ingressroute.yaml
spec:
  routes:
    - match: Host(`hass.yourdomain.com`)
  tls:
    secretName: hass-tls-secret
```

### Trusted Proxies (Required When Behind a Reverse Proxy)

When Home Assistant runs behind an ingress controller or any reverse proxy, you **must** add the proxy's CIDR range to `trusted_proxies` in Home Assistant's `configuration.yaml`. Without this, Home Assistant will log `400 Bad Request` errors and the UI may not load correctly.

Add the following to your Home Assistant `configuration.yaml` (inside the `/config` volume):

```yaml
http:
  use_x_forwarded_for: true
  trusted_proxies:
    - 10.0.0.0/8        # adjust to match your cluster's pod CIDR
    - 172.16.0.0/12
    - 192.168.0.0/16
```

> **Tip:** Run `kubectl get pods -n home-assistant -o wide` to see the pod IP and confirm the correct CIDR range.

---

## Bluetooth / D-Bus Passthrough

To give Home Assistant access to the host's Bluetooth stack, add this patch to the Flux `Kustomization` in `flux/gotk-sync.yaml`:

```yaml
  patches:
    - patch: |
        - op: add
          path: /spec/template/spec/volumes/-
          value:
            name: dbus
            hostPath:
              path: /run/dbus
        - op: add
          path: /spec/template/spec/containers/0/volumeMounts/-
          value:
            name: dbus
            mountPath: /run/dbus
            readOnly: true
      target:
        kind: StatefulSet
        name: home-assistant
```

---

## Accessing Home Assistant

| Service Type | How to access |
|---|---|
| `ClusterIP` (default) | Port-forward: `kubectl port-forward svc/home-assistant 8123:8123 -n home-assistant` then open `http://localhost:8123` |
| `NodePort` | `kubectl get svc -n home-assistant` → open `http://<NODE_IP>:<NODE_PORT>` |
| `LoadBalancer` | Use the external IP from `kubectl get svc -n home-assistant` |
| Traefik `IngressRoute` | Use the hostname configured in `ingressroute.yaml` |

---

## Namespace

All resources deploy into the `home-assistant` namespace, which is created automatically by `namespace.yaml`. To use a different namespace, update the `namespace` field in `kustomization.yaml` and in `flux/gotk-sync.yaml` (`targetNamespace`).

---

## Automatic Updates with Renovate

This repository includes a `renovate.json` configuration. Renovate Bot automatically opens pull requests whenever a new Home Assistant container image is published. The CI workflow (`.github/workflows/tag-on-push.yml`) then creates a matching semver Git tag, which Flux picks up via the `semver: ">=2024.1.1"` selector in `flux/gotk-sync.yaml`.

---

## Useful Commands

```bash
# View logs
kubectl logs -f -l app.kubernetes.io/name=home-assistant -n home-assistant

# Restart the pod (e.g. after editing configuration.yaml)
kubectl rollout restart statefulset/home-assistant -n home-assistant

# Force Flux to reconcile immediately
flux reconcile kustomization home-assistant --with-source

# Remove the deployment
kubectl delete -k .
# or, if using FluxCD, remove home-assistant.yaml from your GitOps repo and push
```

---

## License

This repository is open-source and available under the MIT License.
