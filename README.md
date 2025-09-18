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

It includes `apiVersion` (e.g., `kustomize.config.k8s.io/v1beta1`) and `kind: Kustomization`.

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

### Common Transformers
Common configurations, such as labels (e.g., `org: kodekloud`), are often applied across multiple YAML files (e.g., `deployment.yaml` and `service.yaml`). Manually editing each file is inefficient, time-consuming, and error-prone, especially with many resources.

Kustomize provides **common transformers** to apply changes uniformly across all resources defined in the `kustomization.yaml` file. These include:

- **commonLabels**: Adds labels to all Kubernetes resources.
- **namePrefix** / **nameSuffix**: Adds a prefix or suffix to all resource names.
- **namespace**: Adds a common namespace to all resources.
- **commonAnnotations**: Adds annotations to all resources.

Example in `kustomization.yaml`:
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - service.yaml
commonLabels:
  org: kodekloud
  env: dev
namePrefix: my-app-
namespace: default
commonAnnotations:
  managed-by: kustomize
```

This applies the labels, prefix, namespace, and annotations to all listed resources during the build process.

### Image Transformer
Kustomize's image transformer allows you to customize container images in your manifests without editing the base files. You can override the image name (`newName`), tag (`newTag`), or combine both.

Example in `kustomization.yaml`:
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
images:
  - name: old-image
    newName: new-repo/new-image
    newTag: v1.2.3
```

- **newName**: Replaces the image name (e.g., change repository or registry).
- **newTag**: Updates the image tag (e.g., for versioning).
- **Combination**: Use both `newName` and `newTag` together for full image overrides.

This transformer scans all container images in the resources and applies the specified changes during the build.

### Kustomize Patches
Unlike common transformers, patches provide a more "surgical" approach to targeting one or more specific sections in Kubernetes resources.

Patches require three parameters:
- **Operation Type**: `add`, `remove`, or `replace`.
- **Target**: Specifies the resource to apply the patch to, including `kind`, `version/group`, `name`, `namespace`, `labelSelector`, or `annotationSelector`.
- **Value**: The value to replace or add (required only for `add` or `replace` operations).

There are two main patch methods:
- **JSON 6902 Patch**: Uses a list of operations in JSON format.
- **Strategic Merge Patch**: Uses regular Kubernetes YAML configuration for merging.

#### Types of Patches
1. **Inline**: Defined directly in the same `kustomization.yaml` file.
2. **Separate File**: Created in a separate file (e.g., `patch.yaml`) and referenced in the `patches` section of `kustomization.yaml`.

Example of an inline JSON 6902 patch in `kustomization.yaml`:
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
patches:
  - target:
      kind: Deployment
      name: api-deployment
    patch: |-
      - op: replace
        path: /metadata/name
        value: web-deployment
```

#### Patching Dictionaries
- **Replace Dictionary** (JSON 6902): Specify the operation to replace a key-value pair.
- **Using Strategic Merge Patch**: Copy the relevant section from the original config (e.g., `deployment.yaml`) to `patch.yaml` and update the value.
- **Add Dictionary** (JSON 6902): Use `add` operation to insert a new key-value pair.
- **Add Using Strategic Merge Patch**: Add the new entry in the patch YAML.
- **Remove**: Set the value to `null` (e.g., `org: null`).

#### Patching Lists
To update specific items in a list, mention the index (e.g., `0`, `1`, or `-` for the last item). Strategic merge patches simplify this by allowing you to copy and modify the relevant YAML section in `patch.yaml`.

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
  Alternatively, use native `kubectl` support:
  ```bash
  kubectl apply -k k8s/overlays/dev
  ```

- **Delete Resources**:
  Pipe the output to `kubectl delete` to remove resources:
  ```bash
  kustomize build k8s/overlays/dev | kubectl delete -f -
  ```
  Or use native `kubectl`:
  ```bash
  kubectl delete -k k8s/overlays/dev
  ```

### How It Works
1. The `kustomization.yaml` file in the base directory lists all manifests (e.g., `deployment.yaml`, `service.yaml`).
2. Overlays reference the base directory and specify customizations (e.g., changing replicas or adding resources).
3. The `kustomize build` command combines the base manifests with overlay customizations, applying transformations like `commonLabels` or `patchesStrategicMerge`.
4. The output is a valid Kubernetes manifest that can be applied using `kubectl`.

## Managing Directories with Kustomize

For larger projects with multiple components (e.g., API and database), you can organize manifests into subdirectories under a root directory like `k8s/`.

Without Kustomize, you'd apply each subdirectory separately:
```bash
kubectl apply -f k8s/api/
kubectl apply -f k8s/db/
```

With Kustomize:
- Place a `kustomization.yaml` in the root `k8s/` directory:
  ```yaml
  apiVersion: kustomize.config.k8s.io/v1beta1
  kind: Kustomization
  resources:
    - api/api-depl.yaml
    - api/api-service.yaml
    - db/db-depl.yaml
    - db/db-service.yaml
  ```
- Apply from the root:
  ```bash
  kustomize build k8s/ | kubectl apply -f -
  ```
  Or natively:
  ```bash
  kubectl apply -k k8s/
  ```

As the number of subdirectories grows (e.g., adding cache and Kafka), the root `kustomization.yaml` can become lengthy. To scale:
- Add a `kustomization.yaml` in each subdirectory (e.g., `api/kustomization.yaml`):
  ```yaml
  apiVersion: kustomize.config.k8s.io/v1beta1
  kind: Kustomization
  resources:
    - api-depl.yaml
    - api-service.yaml
  ```
- Update the root `kustomization.yaml` to reference subdirectories:
  ```yaml
  apiVersion: kustomize.config.k8s.io/v1beta1
  kind: Kustomization
  resources:
    - api/
    - db/
    - cache/
    - kafka/
  ```
- Apply as before:
  ```bash
  kustomize build k8s/ | kubectl apply -f -
  ```
  Or:
  ```bash
  kubectl apply -k k8s/
  ```

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
