# Home Assistant Kustomize Deployment

This repository contains a Kustomize setup for deploying [Home Assistant](https://www.home-assistant.io/) on Kubernetes. Kustomize is used to manage and configure the Kubernetes resources needed for Home Assistant. This guide provides instructions on how to deploy Home Assistant using Kustomize with FluxCD.

## Prerequisites

- A Kubernetes cluster (tested with k0s, k3s, and microk8s)
- Kustomize installed ([installation guide](https://kubectl.docs.kubernetes.io/installation/kustomize/))
- FluxCD installed and configured for GitOps ([Flux installation guide](https://fluxcd.io/docs/installation/))
- [Home Assistant](https://www.home-assistant.io/) Docker image available in your container registry (optional)
  
## Overview

The setup includes:

- A Kubernetes Deployment for Home Assistant
- Service definitions

## Configuration

### 1. Clone the Repository

Clone this repository to your local machine:

```bash
    git clone https://github.com/turbo5000c/home-assistant-kustomize.git
    cd home-assistant-kustomize
```

### 2. Update Configuration Files

The following configuration files may need to be adjusted based on your environment and requirements:

- **kustomization.yaml**: Update the image version and namespace as necessary.
- **config.yaml**: Modify Home Assistant configurations as needed.
- **storage.yaml**: Ensure PVC specifications match the storage class available on your cluster.

### 3. Deploying with Kustomize

To apply the configuration with Kustomize, use the following command:
```bash
    kubectl apply -k .
```
This command applies the resources in the current directory, using Kustomize to manage overlays and configurations.

### 4. Deploying with FluxCD

If using FluxCD for GitOps:

1. Ensure your FluxCD setup is configured to watch this repository or a specific directory in it.
2. Add the repository to your Flux configuration by referencing it in your `kustomization.yaml` under the FluxCD setup.
3. Flux will automatically detect changes and apply them to your cluster.

flux kustomization.yaml example:
```yaml
---
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: home-assistant
spec:
  interval: 2h
  ref:
    branch: main
  url: https://github.com/turbo5000c/home-assistant-kustomize.git

---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: home-assistant
spec:
  interval: 4h
  path: "./"
  prune: true
  sourceRef:
    kind: GitRepository
    name: home-assistant
    namespace: home-assistant
  targetNamespace: home-assistant
```

## Namespace Consideration

By default, this setup deploys all resources into the `home-assistant` namespace. If you need to use a different namespace, adjust the `metadata.namespace` field in the `kustomization.yaml` file and make sure the namespace exists on your cluster:

```bash
    kubectl create namespace your-namespace
```


## Storage Configuration

The Persistent Volume Claim (PVC) is configured directly in the StatefulSet manifest. Ensure that the storage class aligns with your cluster’s setup. If necessary, modify the `storageClassName` field in the StatefulSet file to match a compatible storage class available on your cluster.

Additionally, review the storage size in the StatefulSet manifest to make sure it meets your storage requirements. Adjust it by updating the `resources.requests.storage` field in the manifest as needed.

For example, to update the storage class and size, locate the following section in the StatefulSet file:

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

## Accessing Home Assistant

After deployment, Home Assistant should be accessible through the configured Service. By default, the `service.yaml` file is set up with a ClusterIP service type. Modify it to `NodePort` or `LoadBalancer` depending on your access needs.

To access Home Assistant via NodePort:

1. Retrieve the assigned port:
```bash
       kubectl get svc -n home-assistant
```
2. Open a browser and navigate to `http://<NODE_IP>:<PORT>`

For LoadBalancer setups, use the external IP provided by your cloud provider.

## Uninstall

To remove the Home Assistant deployment and its associated resources, run:
```bash
    kubectl delete -k .
```
Or, if using FluxCD, remove the repository or folder from your Flux configuration, and Flux will delete the resources on the cluster.

## Additional Notes

- **Custom Configurations**: Adjust environment variables in `config.yaml` to enable specific Home Assistant features.
- **Logging**: To view logs for Home Assistant, use:
```bash
      kubectl logs -f <pod-name> -n home-assistant
```
Replace `<pod-name>` with the name of the running Home Assistant pod.

## License

This repository is open-source and available under the MIT License.
