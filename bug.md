# Prometheus scrape configurations are deployed unconditionally

**How to categorize this issue?**
/area networking
/kind bug

**What happened**:

The Prometheus scrape configurations for the calico components are deployed unconditionally, even when the respective components are disabled. This can lead to empty Prometheus scrape pools that fail health checks when the gardener/gardener PR [#13341](https://github.com/gardener/gardener/pull/13341) is merged, which validates that all the scrape pools have targets.

Specifically, the bird exporter is disabled by default, but its scrape configuration is nevertheless deployed unconditionally. During a gardener/gardener PR validation, the respective empty scrape pool would report a health check failure. As a mitigation, an exception was added to ignore the empty scrape pool for the `shoot-calico-bird` scrape job. The Prometheus health checks are controlled via a feature gate: they are executed during PR validation, while can remain disabled in real landscapes to allow for collecting and resolving findings like this one, without time pressure.

This issue is about fixing the root cause for the unconditionally deployed scrape configurations in this extension.

Based on a source code review, the bird exporter is not the only affected component. The typha scrape configuration is also deployed unconditionally; however, this goes unnoticed because typha is enabled by default.

If this issue is fixed, the exception in gardener/gardener source code can be removed.

**What you expected to happen**:

The Prometheus scrape configurations should only be deployed when their corresponding components are enabled. When a component is disabled (or disabled after being enabled), the scrape configuration should be removed to prevent empty scrape pools.

**How to reproduce it (as minimally and precisely as possible)**:

1. Create a shoot cluster in the local setup without enabling the bird exporter (default configuration, the `.spec.networking.providerConfig.birdExporter.enabled` property of the `Shoot` resource is not set to `true`)
2. Check the scrape configurations in the shoot's control plane: `kubectl get -n shoot--local--local scrapeconfigs.monitoring.coreos.com | grep calico`
3. Observe that the `shoot-calico-bird` scrape configuration is deployed
4. Check the shoot cluster's services: `kubectl --kubeconfig admin-kubeconf.yaml get svc -n kube-system | grep bird`
5. Observe that no `calico-bird-monitoring` service exists
6. Access the Prometheus UI: `kubectl port-forward -n shoot--local--local prometheus-shoot-0 9090`
7. Observe that the `bird-metrics` scrape job has no targets

If the bird exporter is not enabled, the `shoot-calico-bird` scrape configuration should not be deployed.

**Anything else we need to know?**:

A potential fix should also take the following considerations into account:

1. **Cleanup on upgrade**: When upgrading from an older version of the extension that deployed scrape configs unconditionally, the obsolete resources are not removed automatically because:

<-- Note I also ended each line with a dot -->
- Upgrading the extension doesn't trigger a network resource reconciliation.
- Shoot reconciliation doesn't trigger network resource reconciliation.
- Even when the network resource is reconciled, the extension uses Helm templates to render artifacts but doesn't track which currently deployed artifacts should be removed.

2. **Cleanup on feature disable**: When a previously enabled component is disabled, its scrape configuration is not removed for the same reasons.

- A mechanism to track and remove obsolete artifacts during reconciliation.
- Possibly a cleanup mechanism on extension startup to handle upgrade scenarios.
- Or an alternative approach that doesn't rely on manual resource tracking.

The exception in the gardener/gardener code base is short and straightforward, so it is also valid to close this issue as "won't fix" if a comprehensive solution is not feasible in the near future.

**Environment**:

- Gardener version (if relevant): *not relevant*
- Extension version: `v1.54.0`
- Kubernetes version: *not relevant*
- Cloud provider or hardware configuration: *local setup*

cc @vicwicker
