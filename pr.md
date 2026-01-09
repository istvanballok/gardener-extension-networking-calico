# Deploy the Prometheus Scrape Configurations only when needed

**How to categorize this PR?**
/area networking
/kind enhancement

**What this PR does / why we need it**:

This PR deploys the Prometheus scrape configurations for the calico components (bird, felix and typha) only when the respective components are enabled.

*TODO*:

- [ ] add a mechanism to clean up obsolete scrape configurations when upgrading from an older version of the extension that deployed them unconditionally
- [ ] add a mechanism to clean up obsolete scrape configurations when a previously enabled component is disabled

**Which issue(s) this PR fixes**:

While working on

- https://github.com/gardener/gardener/pull/13341

we noticed that the Prometheus scrape configuration for the calico bird exporter that was introduced in

- https://github.com/gardener/gardener-extension-networking-calico/pull/687

is deployed unconditionally, even when the bird exporter sidecar is not enabled.

This creates empty Prometheus scrape pools that went previously unnoticed. The gardener/gardener PR mentioned above checks for empty scrape pools and a health check task fails if any are found. To prevent these health check failures, this PR adds Helm template conditions to deploy each scrape configuration only when its corresponding component is enabled.

**Special notes for your reviewer**:

cc @vicwicker

Although the bird scrape configuration was the primary concern, the felix and typha scrape configurations are adjusted similarly for consistency.

For some reason, the monitoring artifacts are deployed in a separate internal Helm chart, so as a first step, the relevant chart values from the internal `calico` chart are made available to the internal `calico-monitoring` chart.

<details>
<summary>Instructions for checking this PR in the local setup</summary>

The following section describes how to verify this change in the local setup of Gardener.

*Caveat regarding the processor architecture:* The bird exporter currently uses the `ghcr.io/czerwonk/bird_exporter` image which does not yet support the `arm64` processor architecture. Therefore, the following instructions assume an `amd64` environment.

Start the local setup for Gardener in the `gardener/gardener` repository:

```bash
make kind-up gardener-up
export KUBECONFIG=$PWD/example/gardener-local/kind/local/kubeconfig
k apply -f example/provider-local/shoot.yaml
```

The local setup uses the latest released version of this networking calico extension, so we can observe the issue first. Verify that a Prometheus scrape configuration for the calico bird exporter in the `shoot--local--local` namespace is deployed, although there is no bird exporter sidecar and service in the shoot cluster. This is the issue that this PR aims to fix.

```bash
k get -n shoot--local--local scrapeconfigs.monitoring.coreos.com | grep calico | awk '{print $1}'
```

```text
shoot-calico-bird    *
shoot-calico-felix
shoot-calico-typha
```

```bash
hack/usage/generate-admin-kubeconf.sh > admin-kubeconf.yaml
k --kubeconfig $PWD/admin-kubeconf.yaml get svc -n kube-system | grep calico | awk '{print $1}'
```

```text
calico-felix-monitoring
calico-typha
calico-typha-monitoring
```

Now enable the bird exporter feature in the `Shoot` resource:

```bash
yq '
    .spec.networking.providerConfig.birdExporter.enabled=true
  ' example/provider-local/shoot.yaml \
| k apply -f -
```

and observe that the bird exporter sidecar and service are created in the shoot cluster. The prometheus scrape job for the bird exporter now has a target to scrape and that works as expected.

```bash
k --kubeconfig $PWD/admin-kubeconf.yaml get svc -n kube-system | grep calico | awk '{print $1}'
```

```text
calico-bird-monitoring     *
calico-felix-monitoring
calico-typha
calico-typha-monitoring
```

```bash
k port-forward -n shoot--local--local prometheus-shoot-0 9090
```

```promql
up{job="bird-metrics"}
```

Now reset the `Shoot` resource to disable the bird exporter again. As expected, the superfluous bird exporter scrape configuration is not removed.

```bash
k apply -f example/provider-local/shoot.yaml
```

Next, deploy this PR's version of the networking calico extension into the local setup. First, build the extension images in this `gardener-extension-networking-calico` repository:

```bash
make docker-images
```

Then, push the images to the local registry used by the local setup of Gardener:

```bash
VERSION=$(git rev-parse HEAD)
docker tag europe-docker.pkg.dev/gardener-project/public/gardener/extensions/networking-calico:latest registry.local.gardener.cloud:5001/gardener/extensions/networking-calico/networking-calico:$VERSION
docker push registry.local.gardener.cloud:5001/gardener/extensions/networking-calico/networking-calico:$VERSION
```

Use a patched version of the example controller registration for the calico extension to update the default controller registration of the local setup, to deploy this PR's version of the calico extension:

```bash
VERSION=$VERSION yq '
    (select (.helm) | .helm.values.image.repository) = "registry.local.gardener.cloud:5001/gardener/extensions/networking-calico/networking-calico" |
    (select (.helm) | .helm.values.image.tag) = env(VERSION)
  ' example/controller-registration.yaml \
| k apply --server-side -f -
```

...

</details>

**Release note**:

```other developer
The Prometheus scrape configurations for the calico components are now deployed only when needed, when the respective components that expose the metrics are also deployed. This prevents empty Prometheus scrape pools when certain calico components are disabled.
```
