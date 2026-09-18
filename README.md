# Crossplane Deployment

Helm umbrella chart that deploys and configures [Crossplane](https://www.crossplane.io) through Argo CD, together
with the `Provider`, `ProviderConfig`, and `DeploymentRuntimeConfig` resources needed to manage AWS S3 and OVH
(S3-compatible, via `vshn/provider-minio`) buckets.

## Overview

This chart wraps the upstream [`crossplane`](https://charts.crossplane.io) Helm chart as a dependency and adds:

- `Provider` resources for `provider-aws-s3`, the auto-installed `provider-family-aws` dependency, and
  `provider-minio` (used for OVH).
- `ProviderConfig` resources that wire each provider to its cloud credentials.
- `DeploymentRuntimeConfig` resources to tune CPU/memory for each provider's runtime pod per environment.
- `ExternalSecret` resources that fetch the AWS and OVH credentials referenced by the `ProviderConfig` resources.

> [!NOTE]
> The AWS S3, Hetzner and OVH provider are mutually exclusive per cluster. AWS S3 is enabled by default and
> disabled per environment via `aws.s3.enabled: false`. OVH and Hetzner are only enabled when the respective value block is present.

## Repository Structure

| Path                                | Purpose                                                              |
| ------------------------------------ | --------------------------------------------------------------------|
| `Chart.yaml`                         | Chart metadata and the `crossplane` subchart dependency.             |
| `values.yaml`                        | Default values, shared across all environments.                      |
| `values-local.yaml`                  | Overrides for local development clusters.                            |
| `values-development.yaml`            | Overrides for the development cluster.                               |
| `values-production.yaml`             | Overrides for the production cluster.                                |
| `values-sf-k8s03-dev.yaml`           | Overrides for the `sf-k8s03-dev` cluster (disables AWS, enables Hetzner S3 bucket config). |
| `values-sf-k8s04-dev.yaml`           | Overrides for the `sf-k8s04-dev` cluster (disables AWS, enables OVH S3 bucket config). |
| `values-sf-k8s05-dev.yaml`           | Overrides for the `sf-k8s05-dev` cluster (disables AWS, enables Ionos S3 bucket config). |
| `values-subchart-overrides.yaml`     | Values consumed directly by the `crossplane` subchart. See below.    |
| `templates/`                         | Provider, provider config, runtime config, and secret templates.     |
| `charts/`                            | Vendored `crossplane` subchart dependency archive.                   |
| `tests/`                             | Helm unittest suites and snapshots.                                  |

## Secrets

We use [external secrets](https://external-secrets.io) to manage the secrets needed for this deployment. For
documentation on how to provide these secrets, take a look at the
[external-secrets-deployment](https://github.com/steadforce/external-secrets-deployment) README.

## values-subchart-overrides.yaml

The `values-subchart-overrides.yaml` file is used to override values in the subchart(s) used by this chart. We have
to separate the values for the subcharts from the values for the main chart to be able to unit test for
incompatible changes in values of the subcharts. This is necessary because Helm does not allow switching off the
usage of `values.yaml`. This makes it possible to test whether we use the same registry and repository for images
as the subcharts are using.

## Testing

### Run Helm Unittests

```sh
 docker run \
  --rm \
  -e HELM_CACHE_HOME=/tmp/helm/.config \
  -u $(id -u) \
  -v "$(pwd):/apps" \
  -w /apps \
  helmunittest/helm-unittest .
```

Or with output in JUnit format:

```sh
 docker run \
  --rm \
  -e HELM_CACHE_HOME=/tmp/helm/.config \
  -u $(id -u) \
  -v "$(pwd):/apps" \
  -w /apps \
  helmunittest/helm-unittest \
  -o test-output.xml \
  .
```

> [!TIP]
> If a snapshot test fails after an intentional template change, append `-u` to the `helm-unittest` arguments to
> update the stored snapshots, then review the diff before committing.

### Render Helm Templates Locally

The following command renders the charts the same way Argo CD does for a local deployment:

```sh
 docker run \
  --rm \
  -e HOME=/tmp \
  -u $(id -u) \
  -v "$(pwd):/apps" \
  -w /apps \
  alpine/helm template \
  --include-crds \
  --output-dir _local/local \
  --release-name crossplane \
  --skip-tests \
  -a aws.upbound.io/v1beta1 \
  -a external-secrets.io/v1beta1/ExternalSecret \
  -a pkg.crossplane.io/v1 \
  -a pkg.crossplane.io/v1beta1 \
  -f values-subchart-overrides.yaml \
  -f values-local.yaml \
  -n crossplane-system \
  .
```

You can use this command to check whether the output is as you expect. The `-a` parameters are needed since we use
the Helm feature `.Capabilities.APIVersions.Has` to determine whether a custom resource is installable in the
cluster. Since Helm templating is designed to work offline, we have to list the supported API versions explicitly.
Using `.Capabilities.APIVersions.Has` in templating prevents sync errors in Argo CD if a custom resource can't be
applied because its CRD isn't ready yet.

## Run GitHub Pipeline Locally

This flow runs inside the `SteadOps-Steadies-K8s-Workplace` workbench, which provides `act`, `helm`, `hetzner-k3s`,
`kubectl`, and `yq` directly in its shell. Start the workbench, `cd` into the folder containing this `README.md`,
and run:

```sh
 act
```

On first execution you're asked which flavour of the `act` image should be used. Using the default `medium` is a
good starting point.

## Continuous Integration

Pushes and pull requests trigger reusable workflows from
[`steadforce/steadops-workflows`](https://github.com/steadforce/steadops-workflows):

- `helm-unittest.yaml` runs the suites in `tests/` on every push.
- `trufflehog.yaml` scans the repository for committed secrets on pushes, pull requests, and manual runs.

## Dependency Updates

[Renovate](https://docs.renovatebot.com) keeps the `crossplane` subchart dependency up to date and archives the
updated subchart into `charts/` via the `helmUpdateSubChartArchives` post-update option, see `renovate.json`.

## S3 buckets

For new S3 buckets created via other repos adjust regions depending on provider:
- k8s03 / Hetzner: fsn1
- k8s04 / OVH: de
- k8s05 / Ionos: eu-central-3