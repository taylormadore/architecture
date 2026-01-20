# NN. Registry Proxy Feature Flag Configuration for Hermeto

Date: YYYY-MM-DD

## Status

Proposed

## Context

Hermeto is the content prefetching tool used in Konflux build pipelines to fetch application dependencies before the container image build step. It supports multiple package managers including npm, pip, Go modules, Cargo, Bundler, and Yarn. Currently, Hermeto fetches dependencies directly from upstream package registries such as PyPI, registry.npmjs.org, proxy.golang.org, crates.io, and rubygems.org.

This direct fetching from upstream sources potentially exposes the build process to supply chain attacks. Malicious packages published to public registries can be consumed by builds without any policy evaluation or control. Recent high-profile incidents have demonstrated that attackers actively target package registries to distribute malware.

To address this security gap, organizations can deploy an artifact registry proxy with a policy enforcement layer that:

1. **Evaluates all consumed artifacts** against organizational governance policies during ingestion, automatically quarantining components that violate policy before they are available for use in builds
2. **Stores all consumed artifacts** in a controlled repository, providing an audit trail and enabling reproducible builds

By routing Hermeto's dependency fetching through such a proxy, malicious or non-compliant components can be blocked before they ever reach the build environment.

Following the pattern established in [ADR 0057 - Pipeline Caching Feature Flag Configuration](0057-pipeline-caching-feature-flag.md) for HTTP cache proxies, we need a similar feature flag mechanism for Hermeto's registry proxy support.

**Users** need the ability to:

* Use the registry proxy automatically when available, without manual configuration
* Opt out of registry proxy use for individual pipelines when needed

**Platform administrators** need the ability to:

* Control proxy usage at the cluster level for all pipelines
* Configure proxy URLs for different package managers (npm, pip, cargo, etc.) since they may be served from different proxy repositories
* Apply GitOps processes for this configuration, similar to other Konflux configuration elements

## Decision

Two configuration levels control registry proxy usage:

1. **Cluster-level**: An `allow-hermeto-proxy` flag in the `cluster-config` ConfigMap controls whether the proxy is available cluster-wide. This flag defaults to `"false"`, requiring administrators to explicitly enable the feature.

2. **Pipeline-level**: An `enable-hermeto-proxy` parameter defaults to `"true"`, meaning pipelines will use the proxy when available. Users can set this to `"false"` to opt out.

The proxy is used only when both levels permit it. Since `allow-hermeto-proxy` defaults to `"false"`, the proxy is effectively disabled until an administrator explicitly enables it.

The `cluster-config` ConfigMap in the `konflux-info` namespace will be extended with Hermeto-specific configuration, including proxy URL keys for each supported package manager backend. The ConfigMap will be globally readable from all namespaces in the cluster, but writable only by the infrastructure team and the ArgoCD instance managing the cluster (if any). This is consistent with the existing pattern used for HTTP cache proxy configuration in ADR 0057.

If a package manager's proxy URL is not configured in the ConfigMap (empty or missing), the proxy will not be used for that package manager, even if otherwise enabled.

### Supported Package Managers

The following package manager backends will be supported, each with its own proxy URL configuration:

* **npm** - Node.js package manager
* **yarn** - Yarn modern (v2+)
* **yarn-v1** - Yarn v1.x (classic)
* **gomod** - Go modules (includes separate proxy URL for checksum database)
* **pip** - Python package manager
* **cargo** - Rust package manager
* **bundler** - Ruby package manager

Note: Implementation will be phased, starting with JavaScript package managers (npm, yarn, yarn-v1), with other package managers following in subsequent releases.

### Implementation Details

#### Cluster Configuration

The `cluster-config` ConfigMap in the `konflux-info` namespace will be extended with the following keys:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-config
  namespace: konflux-info
data:
  # ... (existing config) ...

  # Hermeto registry proxy config
  allow-hermeto-proxy: "true"

  # Proxy URLs per package manager
  hermeto-npm-proxy-url: "https://registry-proxy.example.com/npm/"
  hermeto-yarn-proxy-url: "https://registry-proxy.example.com/npm/"
  hermeto-yarn-v1-proxy-url: "https://registry-proxy.example.com/npm/"
  hermeto-gomod-proxy-url: "https://registry-proxy.example.com/go/"
  hermeto-gomod-sum-proxy-url: "https://registry-proxy.example.com/go-sum/"
  hermeto-pip-proxy-url: "https://registry-proxy.example.com/pypi/"
  hermeto-cargo-proxy-url: "https://registry-proxy.example.com/cargo/"
  hermeto-bundler-proxy-url: "https://registry-proxy.example.com/rubygems/"
```

**Notes:**
- Authentication is handled by a reverse proxy layer in front of the registry proxy (not in the ConfigMap)
- Multiple package managers may share the same URL if they use the same upstream registry (e.g., npm, yarn, and yarn-v1 all use registry.npmjs.org)

#### Pipeline Changes

The build pipeline will be modified to:

1. Accept an `enable-hermeto-proxy` parameter (default: `"true"`)
2. Pass it to the init task, which reads the cluster ConfigMap and outputs resolved proxy URLs (or empty strings if disabled)
3. Pass the resolved URLs to the prefetch-dependencies task, which exposes them as `HERMETO_<PKG>__PROXY_URL` environment variables

```yaml
spec:
  params:
    - name: enable-hermeto-proxy
      default: "true"

  tasks:
    - name: init
      params:
        - name: enable-hermeto-proxy
          value: $(params.enable-hermeto-proxy)
      taskRef:
        name: init
      # Reads ConfigMap and outputs resolved URLs:
      #   results:
      #     - name: hermeto-npm-proxy-url
      #     - name: hermeto-yarn-proxy-url
      #     # ... (one per package manager)

    - name: prefetch-dependencies
      params:
        - name: hermeto-npm-proxy-url
          value: $(tasks.init.results.hermeto-npm-proxy-url)
        # ... (remaining URLs wired similarly)
      # Sets env vars: HERMETO_NPM__PROXY_URL, etc.
```

The init task logs the configuration applied (e.g., "Hermeto proxy enabled, using cluster-configured URLs" or "Hermeto proxy disabled by cluster allow-hermeto-proxy=false").

## Consequences

### Positive Outcomes

* **Enhanced Supply Chain Security**: Registry package dependencies can be evaluated against organizational governance policies before being used in builds, with non-compliant or malicious components automatically quarantined
* **Compliance and Audit Trail**: Organizations can demonstrate compliance with security requirements by showing registry dependencies were vetted through a controlled artifact repository
* **Consistent Pattern with ADR 0057**: Uses the same two-level feature flag pattern as HTTP cache proxy, making it familiar to users and administrators. Note: `allow-hermeto-proxy` defaults to `"false"` (unlike the cache proxy) to require explicit administrator opt-in for this feature.
* **Improved User Experience**: Users can control proxy usage through simple boolean flags without needing to understand proxy URLs, authentication mechanisms, or repository configurations
* **Cluster-level Control**: The cluster-level `allow-hermeto-proxy` flag gives administrators explicit control over proxy availability, requiring conscious opt-in and allowing quick cluster-wide disable in emergency situations
* **GitOps Integration**: All configuration is managed through declarative ConfigMaps and pipeline parameters, consistent with GitOps practices
* **Per-Package-Manager Flexibility**: Administrators can enable proxies for some package managers while leaving others direct (e.g., enable npm proxy while pip proxy repository is still being implemented)

### Negative Outcomes and Trade-offs

* **Implementation Complexity**: The init task becomes more complex as it needs to:
  - Resolve configuration from multiple sources (pipeline params, cluster ConfigMap)
  - Apply precedence rules for enable/allow flags
  - Handle and emit many new package manager proxy URLs

  This complexity increases the maintenance burden and potential for configuration errors.

* **Parameter Proliferation**: This feature adds 8 new result parameters to the init task output (one URL per package manager). While necessary for flexibility, this contributes to growing complexity in the pipeline task wiring and makes the pipeline YAML more verbose.

* **Configuration Overhead**: Administrators must configure and maintain proxy URLs for each package manager in the cluster ConfigMap. While these URLs should be stable, incorrect configuration can break hermetic builds cluster-wide. Administrators need to understand:
  - Registry proxy URL conventions
  - Which package managers share repository groups (e.g., npm/yarn/yarn-v1)
  - How to test proxy configurations before deploying them

* **Migration Impact**: Since `allow-hermeto-proxy` defaults to `"false"`, existing pipelines are not affected until an administrator explicitly enables the feature. When enabling, administrators should:
  - Configure and test proxy URLs before setting `allow-hermeto-proxy: "true"`
  - Communicate the change to users before enabling cluster-wide
  - Be aware that users can opt out by setting `enable-hermeto-proxy: "false"` in their pipeline definitions

## Alternatives Considered

### Single Base URL Approach

An alternative approach was considered where only a single base URL would be configured (e.g., `https://registry-proxy.example.com`), with per-package-manager paths constructed at runtime. This could be either through Tekton parameter concatenation or within the prefetch task script.

**Advantages:**
- Fewer parameters to configure and pass through the pipeline
- Simpler cluster ConfigMap (1 URL instead of 8)
- Easier to maintain when all package managers use the same proxy server

**Disadvantages:**
- **Loss of per-package-manager flexibility**: Administrators cannot selectively enable proxies for some package managers while leaving others disabled
- **Hardcoded path assumptions**: The path structure (e.g., `/npm/`, `/pypi/`) is configuration data that would need to be hardcoded in the prefetch-dependencies task
- **All-or-nothing**: Cannot accommodate environments where different package managers use different proxy servers

## References

* [ADR 0057 - Pipeline Caching Feature Flag Configuration](0057-pipeline-caching-feature-flag.md)
