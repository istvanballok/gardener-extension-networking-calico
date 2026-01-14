# Prometheus scrape configurations are deployed unconditionally

**How to categorize this issue?**
/area networking
/kind bug

**What happened**:

The Prometheus scrape configurations for the calico components are deployed unconditionally, even when the respective components are disabled. This can lead to empty Prometheus scrape pools that fail the new Prometheus health checks that are introduced with the gardener/gardener PR [#13341](https://github.com/gardener/gardener/pull/13341).

Specifically, the bird exporter is disabled by default, but its scrape configuration is nevertheless deployed unconditionally. During a gardener/gardener PR validation, the respective empty scrape pool would report a health check failure. As a mitigation, an exception was added to ignore the empty scrape pool for the `shoot-calico-bird` scrape job. The Prometheus health checks are controlled via a feature gate: they are enabled during PR validation. They shall remain disabled in production landscapes, until findings like this one are collected and resolved without time pressure.

This issue is about fixing the root cause for the unconditionally deployed scrape configurations in this extension.

Based on a source code review, the bird exporter is not the only affected component. The typha scrape configuration is also deployed unconditionally; however, this goes unnoticed because typha is enabled by default.

If this issue is fixed, the exception in gardener/gardener source code can be removed.

**What you expected to happen**:

The Prometheus scrape configurations should only be deployed when their corresponding components are enabled. When a component is disabled (or disabled after being enabled), the scrape configuration should be removed to prevent empty scrape pools.

**How to reproduce it (as minimally and precisely as possible)**:

1. Create a shoot cluster in the local setup without enabling the bird exporter (the default configuration)
   - The `.spec.networking.providerConfig.birdExporter.enabled` property of the `Shoot` resource is not set to `true`.
2. Check the scrape configurations in the shoot's control plane: `kubectl get -n shoot--local--local scrapeconfigs.monitoring.coreos.com | grep calico`.
   - Observe that the `shoot-calico-bird` scrape configuration is deployed.
3. Check the shoot cluster's services:
   - `hack/usage/generate-admin-kubeconf.sh > admin-kubeconf.yaml`
   - `kubectl --kubeconfig admin-kubeconf.yaml get svc -n kube-system | grep bird`
   - Observe that no `calico-bird-monitoring` service exists.
4. Access the Prometheus UI: `kubectl port-forward -n shoot--local--local prometheus-shoot-0 9090`.
   - Observe that the `bird-metrics` scrape job has no targets.

If the bird exporter is not enabled, the `shoot-calico-bird` scrape configuration should not be deployed.

**Anything else we need to know?**:

A potential fix should also take the following considerations into account:

1. *Cleanup on extension upgrade*: When upgrading from an older version of the extension that deployed scrape configs unconditionally, the obsolete resources should be removed automatically during the next regular reconciliation of the network resource. This ensures that there will be no empty scrape pools soon after the extension upgrade.

2. *Cleanup on feature disable*: When a previously enabled component is disabled, its scrape configuration should be removed together with the component itself. This ensures that no empty scrape pools will be left behind after a component is disabled.

The mitigation in the gardener/gardener code base is short and straightforward, so it is also valid to close this issue as "won't fix" if a comprehensive solution is not feasible in the near future.

**Environment**:

- Gardener version (if relevant): *not relevant*
- Extension version: `v1.54.0`
- Kubernetes version: *not relevant*
- Cloud provider or hardware configuration: *local setup*

cc @vicwicker
