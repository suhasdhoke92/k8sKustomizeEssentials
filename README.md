# K8sKustomizeEssentials

This repository provides a comprehensive guide to Kustomize, a Kubernetes-native configuration management tool. It covers Kustomize's core concepts, problem statement, installation, usage, and comparison with Helm to help you manage Kubernetes manifests efficiently.

## What is Kustomize?

Kustomize is a tool for customizing Kubernetes resource configurations in a declarative, YAML-based manner. It allows you to manage environment-specific configurations (e.g., development, staging, production) without duplicating manifests, making it a scalable solution for Kubernetes deployments.

### Problem Statement & Ideology
Managing Kubernetes configurations across multiple environments (e.g., dev, staging, prod) often involves creating separate directories for each environment, with duplicated manifests modified for attributes like replicas (e.g., 1 for dev, 2 for staging, 5 for prod). This approach is error-prone and not scalable.

Kustomize solves this by using a **base** configuration and **overlays** to customize resources per environment. The base directory contains the core manifests, while overlays define environment-specific changes or additional resources. Kustomize generates the final manifests, which are then applied to a Kubernetes cluster.

Key features:
- **Base Directory**: Contains common Kubernetes manifests (e.g., `deployment.yaml`, `service.yaml`).
- **Overlays**: Environment-specific directories (e.g., `overlays/dev`, `overlays/staging`, `overlays/prod`) that override base configurations or add new resources.
- **Built-in Feature**: Kustomize is integrated into `kubectl` (since Kubernetes 1.14), requiring no additional package installation.

Kustomize generates valid YAML manifests, which can be piped to `kubectl apply` for deployment.

## Kustomize vs Helm

Kustomize and Helm both address the challenge of managing environment-specific Kubernetes configurations, but they differ in approach and features:

- **Helm**:
  - Uses Go templating to assign variables to properties (e.g., `{{ .Values.replicaCount }}`) defined in a `values.yaml` file (e.g., `replicaCount: 1`).
  - Acts as a package manager, enabling chart distribution and versioning.
  - Supports advanced features like conditionals, loops, functions, and hooks.
  - Templates are not valid YAML due to Go templating syntax, which can be harder to read and debug.
- **Kustomize**:
  - Uses plain YAML for both base and overlay configurations, making it easier to read and maintain.
  - Focuses solely on configuration customization, not package management.
  - Simpler and more lightweight, avoiding complex templating logic.
  - Built into `kubectl`, requiring no separate installation for modern Kubernetes versions.

**When to Use**:
- Use Kustomize for straightforward, YAML-based configuration management.
- Use Helm for complex applications requiring packaging, versioning, and advanced templating.

## Installation

Kustomize is built into `kubectl` (since Kubernetes 1.14), but you can install the standalone CLI for advanced usage. The installation script automatically selects the appropriate version for your operating system (Windows, Linux, or macOS).

Install Kustomize CLI:
```bash
curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh" | bash
```

Verify the installation:
```bash
kustomize version --short
```

Move the binary to a directory in your PATH (e.g., `/usr/local/bin`):
```bash
sudo mv kustomize /usr/local/bin/
```

## Kustomization.yaml File

The `kustomization.yaml` file is the core of Kustomize's configuration management. It must be named `kustomization.yaml` or `kustomization.yml` and defines:
- A list of Kubernetes manifests to manage.
- Customizations to apply (e.g., overriding replicas, adding labels).

Example `kustomization.yaml` in the base directory:
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - service.yaml
commonLabels:
  app: my-app
```

Example overlay `kustomization.yaml` (e.g., `overlays/dev/kustomization.yaml`):
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
bases:
  - ../../base
patchesStrategicMerge:
  - patch.yaml
```

Example `patch.yaml` for dev environment:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 1
```

### Key Commands
- **Build Manifests**:
  Generate the final manifests by combining the base and overlay configurations:
  ```bash
  kustomize build k8s/overlays/dev
  ```
  The `kustomize build` command does not apply resources to the cluster; it outputs the merged YAML.

- **Apply to Cluster**:
  Pipe the output to `kubectl apply` to deploy the resources:
  ```bash
  kustomize build k8s/overlays/dev | kubectl apply -f -
  ```

### How It Works
1. The `kustomization.yaml` file in the base directory lists all manifests (e.g., `deployment.yaml`, `service.yaml`).
2. Overlays reference the base directory and specify customizations (e.g., changing replicas or adding resources).
3. The `kustomize build` command combines the base manifests with overlay customizations, applying transformations like `commonLabels` or `patchesStrategicMerge`.
4. The output is a valid Kubernetes manifest that can be applied using `kubectl`.

## Example Directory Structure
```
k8s/
├── base/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── kustomization.yaml
└── overlays/
    ├── dev/
    │   ├── patch.yaml
    │   └── kustomization.yaml
    ├── staging/
    │   ├── patch.yaml
    │   └── kustomization.yaml
    └── prod/
        ├── patch.yaml
        └── kustomization.yaml
```

## Conclusion

Kustomize provides a simple, YAML-based approach to managing Kubernetes configurations across multiple environments. By using a base configuration and environment-specific overlays, it eliminates the need for duplicated manifests, making it scalable and easy to maintain. Compared to Helm, Kustomize is lightweight and focuses solely on configuration customization, leveraging plain YAML for readability and simplicity. As a built-in Kubernetes feature, it’s an excellent choice for teams seeking a straightforward solution for managing manifests.
