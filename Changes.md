# Changes Summary

This document summarizes the recent changes to the Helm chart, introducing `overrideContainer` for more flexible container configuration, detailing the new Kubernetes Gateway API routes feature, and explaining the `customServiceAccount` and `serviceWatcher` configurations.

## New Features

### 1. `overrideContainer` for Unified Container Overrides

This new feature provides a unified way to override container properties such as `image`, `env` (environment variables), and `resources` (CPU/memory limits and requests) for specific containers within your deployment. This simplifies managing container-specific configurations without directly modifying the main `values.yaml` or needing separate override fields for each property.

**Key Behavior:**
*   If a property (image, environment variable, or resource) is defined in both the container's original definition and `overrideContainer`, the value from `overrideContainer` takes precedence.
*   For environment variables, the logic ensures no duplicate variables in the final container definition; the original is filtered out if overridden.
*   For resources, `overrideContainer` values are merged with existing resources, with `overrideContainer` taking precedence.

**Usage Example in `values.yaml` (or an override file):**

```yaml
overrideContainers:
  my-app-container: # Replace 'my-app-container' with your container's name
    image: "my-registry/my-app:v2.0.0"
    env:
      DEBUG_MODE: "true"
      API_ENDPOINT: "https://api.example.com/v2"
    resources:
      limits:
        cpu: "500m"
        memory: "1Gi"
      requests:
        cpu: "250m"
        memory: "512Mi"
  another-container:
    image: "another-registry/another-app:latest"
    env:
      LOG_LEVEL: "info"
    resources:
      limits:
        cpu: "1"
        memory: "2Gi"
      requests:
        cpu: "500m"
        memory: "1Gi"
```

### 2. Kubernetes Gateway API Routes

This Helm chart now supports defining Kubernetes Gateway API `HTTPRoute` resources directly through `values.yaml`, offering a modern alternative to traditional Ingress for managing external access to services. This feature allows for more advanced traffic management capabilities.

**Usage Example in `values.yaml` (or an override file):**

```yaml
gateway:
  name: "gogateway-ai-gateway" # Name of the Gateway resource to attach routes to

routes:
  - name: "example"
    enabled: true
    hostnames:
      - "dev.gogateway.ai"
      - "*.dev.gogateway.ai"
    configMapHostnames:
      enabled: false # Set to true to fetch hostnames from a ConfigMap
      name: "gateway-env"
      keys: # List of keys within the ConfigMap containing comma-separated hostnames
        - "GATEWAY_DOMAIN_NAME"
        - "GATEWAY_WILDCARD_DOMAIN_NAME"
    matches:
      - type: PathPrefix
        value: "/newui"
    healthCheck:
      checkIntervalSec: 15
      timeoutSec: 10
      unhealthyThreshold: 3
      healthyThreshold: 1
      logConfig:
        enabled: true
      httpHealthCheck:
        requestPath: "/newui"
  # You can define multiple routes here
  # - name: "another-route"
  #   enabled: false
  #   hostnames:
  #     - "another.dev.gogateway.ai"
  #   matches:
  #     - type: PathPrefix
  #       value: "/api"
  #   healthCheck:
  #     checkIntervalSec: 30
  #     timeoutSec: 15
  #     unhealthyThreshold: 5
  #     healthyThreshold: 2
  #     logConfig:
  #       enabled: false
  #     httpHealthCheck:
  #       requestPath: "/healthz"
```

### 3. `customServiceAccount` for Custom Service Accounts

The `customServiceAccount` configuration allows you to create and manage a dedicated Kubernetes Service Account for your deployment. This is useful when your application requires specific permissions that are not covered by the default service account, or when you need to integrate with external identity providers.

**When to use:**
*   Your application needs specific RBAC permissions (e.g., to access Kubernetes API resources).
*   You are integrating with external identity management systems that require a dedicated service account.
*   You want to follow the principle of least privilege by granting only necessary permissions to your application.

**Usage Example in `values.yaml` (or an override file):**

```yaml
customServiceAccount:
  create: true
  name: my-custom-service-account
```

### 4. `serviceWatcher` for Service Monitoring and Role Bindings

The `serviceWatcher` feature enables the deployment of a service watcher component that can monitor Kubernetes services and create necessary Role and RoleBinding resources. This is particularly useful for applications that need to dynamically discover and interact with other services within the cluster, or when you need to grant specific permissions to the service watcher itself.

**When to use:**
*   Your application needs to discover and interact with other services in the cluster (e.g., a proxy or a service mesh sidecar).
*   You need to grant specific permissions to a service account to watch and manage Kubernetes resources (e.g., services, deployments, pods).
*   You are implementing a custom controller or operator that requires cluster-wide visibility or control over certain resource types.

**Usage Example in `values.yaml` (or an override file):**

```yaml
serviceWatcher:
  enabled: true
  serviceAccountName: my-custom-service-account # Use the custom service account created above
  role:
    name: service-watcher-role
    apiGroups:
      - ""
      - apps
      - batch
    resources:
      - services
      - deployments
      - pods
  roleBinding:
    name: service-watcher-rolebinding
```

## Implementation Details

*   **`values.yaml`**:
    *   The `overrideEnv` and `overrideResources` fields have been replaced by a new unified field `overrideContainers`.
    *   New `gateway` and `routes` configuration blocks have been added to support Kubernetes Gateway API.
    *   New `customServiceAccount` and `serviceWatcher` configuration blocks have been added.
    *   Comments have been updated to reflect these changes and provide guidance.

*   **`templates/deployment.yaml`**:
    *   The templating logic for `containers` and `initContainers` has been updated to check for `$.Values.overrideContainers`.
    *   When `overrideContainers` is present for a specific container, its `image`, `env`, and `resources` values are merged with the container's existing definitions, with `overrideContainers` taking precedence.

*   **`templates/routes.yaml`**:
    *   This file would contain the logic to generate `HTTPRoute` resources based on the `$.Values.routes` configuration.

*   **`templates/serviceaccount.yaml`**: 
    *   This file would contain the logic to generate `ServiceAccount` resources based on the `$.Values.customServiceAccount` configuration.

*   **`templates/service-watcher.yaml`**: 
    *   This file would contain the logic to generate `Role` and `RoleBinding` resources based on the `$.Values.serviceWatcher` configuration.

*   **`../../gateway-dev/gatewaycore/devspace-common.yaml`**:
    *   The `deploy-with-gitops` function now supports multiple `--values-template` flags.
    *   Each specified `values-template` file will be passed to the Helm upgrade command using separate `--values` flags.
    *   Note: Image tag substitution and environment variable substitution in values files are currently skipped when using multiple values templates.

**Usage Example for multiple `values-template` files in `devspace.yaml`:**

```yaml
pipelines:
  deploy:
    run: |-
      deploy-with-gitops --chart https://github.com/gogatewayai/gw-component-chart.git#master \
                         --release my-app \
                         --namespace default \
                         --values-template ./gitops/base-values.yaml \
                         --values-template ./gitops/environment-override-values.yaml \
                         --helm-args "--wait --atomic"