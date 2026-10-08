# Crossplane Deployment

Helm umbrella chart that deploys and configures [Crossplane](https://www.crossplane.io) through Argo CD, together
with the `Provider`, `ProviderConfig`, and `DeploymentRuntimeConfig` resources needed to manage AWS S3 buckets and
S3-compatible Hetzner and OVH buckets (via `vshn/provider-minio`).

## Overview

The chart depends on the upstream [`crossplane`](https://charts.crossplane.io/stable) chart (version pinned in
`Chart.yaml` and `Chart.lock`) and adds the resources Steadforce needs on top of it:

| Template | Resource |
| --- | --- |
| `namespace.yaml` | `Namespace` of the release |
| `aws-s3-provider.yaml` | `Provider` for `upbound/provider-aws-s3` |
| `aws-family-provider.yaml` | `Provider` `upbound-provider-family-aws`, with the same tag as the S3 provider |
| `aws-s3-provider-runtime-config.yaml` | `DeploymentRuntimeConfig` with the S3 provider's resources |
| `aws-s3-provider-config.yaml` | `ProviderConfig` `aws-s3` that assumes the stage's IAM role |
| `aws-external-secret.yaml` | `ExternalSecret` `aws` with the AWS credentials of the stage |
| `minio-s3-provider.yaml` | `Provider` for `vshn/provider-minio`, used for Hetzner and OVH |
| `minio-s3-provider-runtime-config.yaml` | `DeploymentRuntimeConfig` with the minio provider's resources |
| `hetzner-s3-provider-config.yaml` | `ProviderConfig` `hetzner-s3` for the Hetzner S3 endpoint |
| `hetzner-external-secret.yaml` | `ExternalSecret` `hetzner` with the Hetzner S3 credentials |
| `ovh-s3-provider-config.yaml` | `ProviderConfig` `ovh-s3` for the OVH S3 endpoint |
| `ovh-external-secret.yaml` | `ExternalSecret` `ovh` with the OVH S3 credentials |
| `default-provider-runtime-config.yaml` | `default` `DeploymentRuntimeConfig` with default provider resources |

The family provider is a dependency of `provider-aws-s3` that Crossplane installs automatically but never upgrades.
The chart adopts it under the name Crossplane gives it, so both providers stay on the same version.

> [!NOTE]
> Each cluster uses only one of the AWS S3, Hetzner, and OVH providers; the chart does not enforce this. AWS S3 is
> enabled by default and disabled per cluster with `aws.s3.enabled: false`. Hetzner and OVH are only enabled when
> the `hetzner` or `ovh` value block is present, and both share the minio provider.

All templates except the namespace are guarded by `.Capabilities.APIVersions.Has`. They are rendered only when the
cluster (or `helm template -a`) reports the API version of the custom resource. This prevents Argo CD sync errors
while a CRD is not ready yet.

`values.yaml` sets these top-level keys:

- `global.stage`: stage name (`local`), used in the secret keys `/<stage>/crossplane/...`.
- `aws.s3`: whether the AWS S3 resources are enabled and the IAM role the `aws-s3` `ProviderConfig` assumes.
- `crossplane`: overrides for the upstream chart (memory request of the Crossplane container).
- `providers.awsS3`, `providers.minio`, and `providers.default`: provider images and runtime config resources.

### Secrets

We use [External Secrets](https://external-secrets.io) to manage the secrets needed for this deployment. The
`ExternalSecret` resources read these keys from the `ClusterSecretStore` `awssm-parameter-store`:

| Secret | Key | Properties |
| --- | --- | --- |
| `aws` | `/<stage>/crossplane/aws` | whole value as `creds` |
| `hetzner` | `/<stage>/crossplane/hetzner` | `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY` |
| `ovh` | `/<stage>/crossplane/ovh-minio` | `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY` |

For how to provide these secrets, see the
[external-secrets-deployment](https://github.com/steadforce/external-secrets-deployment) README.

## Prerequisites

- [Docker](https://docs.docker.com/get-docker/) for the containerized commands, or
- the `SteadOps-Steadies-K8s-Workplace` workbench, which provides `helm`, `yq`, `kubectl`, `hetzner-k3s`, and `act`.

All commands run from the repository root.

## Repository Layout

| Path | Purpose |
| --- | --- |
| `Chart.yaml` | Chart metadata and the `crossplane` dependency |
| `Chart.lock` | Locked dependency version, committed |
| `charts/` | Committed subchart archive, kept in sync with `Chart.lock` |
| `templates/` | Provider, runtime config, provider config, secret, and namespace templates |
| `values.yaml` | Defaults (stage `local`) |
| `values-development.yaml` | Development stage |
| `values-production.yaml` | Production stage and IAM role |
| `values-local.yaml` | Local cluster: CPU limits and requests set to `0m`, memory requests to `0Mi` |
| `values-sf-k8s03-dev.yaml` | `sf-k8s03-dev` cluster: disables AWS S3, sets the Hetzner S3 endpoint |
| `values-sf-k8s04-dev.yaml` | `sf-k8s04-dev` cluster: disables AWS S3, sets the OVH S3 endpoint |
| `values-subchart-overrides.yaml` | Overrides for subchart values (see [Testing](#testing)) |
| `tests/` | helm-unittest suites; snapshots in `tests/__snapshot__/` are gitignored |
| `renovate.json` | Renovate configuration |
| `.github/workflows/` | CI workflows |

The unit tests apply the cluster files `values-sf-k8s03-dev.yaml` and `values-sf-k8s04-dev.yaml` on top of
`values-development.yaml`.

## Setup

The subchart archive in `charts/` is committed, so a fresh clone renders and tests without further setup. To restore
`charts/` from `Chart.lock`, for example after deleting the archive, build the dependencies.

In the workbench:

```sh
 yq 'explode(.) | .dependencies[] | select(.repository == "http*") | .name + " " + .repository' Chart.yaml |
   while read -r name repo; do helm repo add --force-update "$name" "$repo"; done
 helm dependency build .
```

With Docker:

```sh
 docker run \
   -e HOME=/tmp \
   --entrypoint sh \
   --rm \
   -u $(id -u) \
   -v "$(pwd):/apps" \
   -w /apps \
   alpine/helm -c '
     yq "explode(.) | .dependencies[] | select(.repository == \"http*\") | .name + \" \" + .repository" Chart.yaml |
       while read -r name repo; do helm repo add --force-update "$name" "$repo"; done &&
     helm dependency build .
   '
```

## Rendering

The following command renders the chart like Argo CD does for the local cluster and writes the manifests to
`_local/local` (gitignored). The `-a` flags list the custom resource API versions the templates check with
`.Capabilities.APIVersions.Has`; Helm templating works offline, so they must be passed explicitly.

In the workbench:

```sh
 helm template crossplane . \
   -a aws.upbound.io/v1beta1 \
   -a external-secrets.io/v1/ExternalSecret \
   -a minio.crossplane.io/v1 \
   -a pkg.crossplane.io/v1 \
   -a pkg.crossplane.io/v1beta1 \
   -f values-subchart-overrides.yaml \
   -f values-local.yaml \
   --include-crds \
   -n crossplane-system \
   --output-dir _local/local \
   --skip-tests
```

With Docker:

```sh
 docker run \
   -e HOME=/tmp \
   --rm \
   -u $(id -u) \
   -v "$(pwd):/apps" \
   -w /apps \
   alpine/helm template crossplane . \
   -a aws.upbound.io/v1beta1 \
   -a external-secrets.io/v1/ExternalSecret \
   -a minio.crossplane.io/v1 \
   -a pkg.crossplane.io/v1 \
   -a pkg.crossplane.io/v1beta1 \
   -f values-subchart-overrides.yaml \
   -f values-local.yaml \
   --include-crds \
   -n crossplane-system \
   --output-dir _local/local \
   --skip-tests
```

For the other stages, replace `values-local.yaml` with `values-development.yaml` or `values-production.yaml` and
adjust the output directory. For the `sf-k8s03-dev` or `sf-k8s04-dev` cluster, pass its cluster file after
`-f values-development.yaml`.

## Testing

### Subchart Overrides

The `values-subchart-overrides.yaml` file overrides values of the subchart(s) used by this chart. The values for
the subcharts are kept apart from the values of the main chart so the unit tests detect incompatible changes in the
subchart values. This is necessary because Helm does not allow switching off `values.yaml`. It also makes it possible
to test that the chart uses the same image registry and repository as the subcharts.

### Run Helm Unittests

```sh
 docker run \
   -e HELM_CACHE_HOME=/tmp/helm/.config \
   --rm \
   -u $(id -u) \
   -v "$(pwd):/apps" \
   -w /apps \
   helmunittest/helm-unittest .
```

helm-unittest writes no report file by default. For a JUnit report like the CI produces, add
`-t JUnit -o test-output.xml` before the chart path; `test-output.xml` is gitignored. Without `-t JUnit`, `-o`
writes XUnit.

In the workbench, run `helm unittest .` directly. The workbench image does not ship the helm-unittest plugin; this
only works because the workbench mounts `$HOME`, so a plugin installed in the host's Helm home is available. Install
it once with:

```sh
 helm plugin install --verify=false https://github.com/helm-unittest/helm-unittest.git
```

Helm 4 needs `--verify=false` for this unsigned plugin, as the pipeline does.

> [!TIP]
> If a snapshot test fails after an intentional template change, add `-u` before the chart path to update the
> stored snapshots. Snapshots live in the gitignored `tests/__snapshot__/` and are rebuilt locally, so they catch
> side effects but prove nothing on their own; behavior that must not change belongs in a direct assertion.

## CI/CD

| Workflow | Trigger | What it does |
| --- | --- | --- |
| `helm-unittest.yaml` | every push | Calls `helm-unittest.yaml@v4.2.0` of `steadforce/steadops-workflows` |
| `trufflehog.yaml` | push/PR to `main`, manual | Calls `trufflehog-oss.yaml@v4.2.0`, scans the pushed commit range |

The reusable helm-unittest workflow runs every chart that has a `tests/` directory. It installs the dependencies
with `helm dependency build`, pinned by the committed `Chart.lock` (it falls back to `helm dependency update` with a
warning when no lock file exists). It then runs `helm unittest` including the subchart tests, publishes the JUnit
results, and runs `helm lint`.

On branches starting with `renovate/`, the workflow posts the result to MS Teams:

- Successes go to the `STEADOPS_HELM_RENOVATION_MS_TEAMS_WEBHOOK` secret.
- Failures go to the `STEADOPS_HELM_RENOVATION_ERROR_MS_TEAMS_WEBHOOK` secret, a separate channel for broken
  Renovate updates. When that secret is not set, failures fall back to `STEADOPS_HELM_RENOVATION_MS_TEAMS_WEBHOOK`.

Both secrets are optional; without them no notification is sent.

> [!NOTE]
> The repository has no hydration workflow. Argo CD renders the chart itself.

### Run the GitHub Pipeline Locally

To run the GitHub workflows locally, start the workbench and run `act` from the repository root:

```sh
 act
```

On the first run, `act` asks which flavour of the act image to use; the default `medium` is a good starting point.
Under `act`, the unit test workflow skips publishing the test results and sending notifications.

## Dependency Updates

[Renovate](https://docs.renovatebot.com) keeps the dependencies up to date. `renovate.json` extends
`config:recommended` and `:dependencyDashboard`, so updates arrive as pull requests without automerge, and the
`helmUpdateSubChartArchives` post-update option also refreshes the committed archive in `charts/`.

To update the `crossplane` chart manually, change its version in `Chart.yaml` and regenerate `Chart.lock` and the
archive in `charts/`:

```sh
 docker run \
   -e HOME=/tmp \
   --rm \
   -u $(id -u) \
   -v "$(pwd):/apps" \
   -w /apps \
   alpine/helm dependency update .
```

Helm replaces the old archive in `charts/`. Commit the new `Chart.lock` and archive together with `Chart.yaml`.
