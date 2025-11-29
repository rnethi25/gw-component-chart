# Changes Summary

This document summarizes the recent changes to the Helm chart, introducing `overrideEnv` and `overrideResources` for more flexible container configuration, detailing the new Kubernetes Gateway API routes feature, and explaining the `customServiceAccount` and `serviceWatcher` configurations.

## New Features

### 1. `overrideEnv` for Environment Variable Overrides

Similar to how `extraEnv` was used, `overrideEnv` allows you to override or add environment variables for specific containers within your deployment. This provides a clean way to manage container-specific environment configurations without modifying the main `values.yaml` directly.

**Key Behavior:**
*   If an environment variable is defined in both the container's `env` list and `overrideEnv`, the value from `overrideEnv` takes precedence.
*   The chart logic ensures that there are no duplicate environment variables in the final container definition. It filters out the original variable if it's being overridden.

**Usage Example in `values.yaml` (or an override file):**

```yaml
overrideEnv:
  my-app-container: # Replace 'my-app-container' with your container's name
    DEBUG_MODE: "true"
    API_ENDPOINT: "https://api.example.com/v2"
  another-container:
    LOG_LEVEL: "info"
```

### 2. `overrideResources` for Container Resource Overrides

This new feature enables overriding resource `limits` and `requests` for individual containers. This is particularly useful for fine-tuning resource allocation for specific containers in different environments (e.g., development, staging, production) without altering the base chart.

**Usage Example in `values.yaml` (or an override file):**

```yaml
overrideResources:
  my-app-container: # Replace 'my-app-container' with your container's name
    limits:
      cpu: "500m"
      memory: "1Gi"
    requests:
      cpu: "250m"
      memory: "512Mi"
  another-container:
    limits:
      cpu: "1"
      memory: "2Gi"
    requests:
      cpu: "500m"
      memory: "1Gi"
```

### 3. Kubernetes Gateway API Routes

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

### 4. `customServiceAccount` for Custom Service Accounts

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

### 5. `serviceWatcher` for Service Monitoring and Role Bindings

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
    *   The `extraEnv` field has been renamed to `overrideEnv`.
    *   A new field `overrideResources` has been added.
    *   New `gateway` and `routes` configuration blocks have been added to support Kubernetes Gateway API.
    *   New `customServiceAccount` and `serviceWatcher` configuration blocks have been added.
    *   Comments have been updated to reflect these changes and provide guidance.

*   **`templates/deployment.yaml`**:
    *   The templating logic for `containers` and `initContainers` has been updated to check for `$.Values.overrideEnv` and `$.Values.overrideResources`.
    *   When `overrideEnv` is present, it is merged with the container's existing `env`. The logic ensures that if a variable exists in both, the one from `overrideEnv` is used, and the original is removed to prevent duplicates.
    *   When `overrideResources` is present, its values are merged with the container's existing `resources`, with `overrideResources` taking precedence.

*   **`templates/routes.yaml`**: (Assumed to exist or be created for this feature)
    *   This file would contain the logic to generate `HTTPRoute` resources based on the `$.Values.routes` configuration.

*   **`templates/serviceaccount.yaml`**: 
    *   This file would contain the logic to generate `ServiceAccount` resources based on the `$.Values.customServiceAccount` configuration.

*   **`templates/service-watcher.yaml`**: 
    *   This file would contain the logic to generate `Role` and `RoleBinding` resources based on the `$.Values.serviceWatcher` configuration.