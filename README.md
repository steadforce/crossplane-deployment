# Crossplane Deployment

Helm umbrella chart that deploys [Crossplane](https://www.crossplane.io) with the AWS S3 provider through Argo CD.

## Overview

The chart depends on the upstream [`crossplane`](https://charts.crossplane.io/stable) chart (version pinned in
`Chart.yaml` and `Chart.lock`) and adds the resources Steadforce needs on top of it:

| Template | Resource |
| --- | --- |
| `namespace.yaml` | `Namespace` of the release |
| `aws-s3-provider.yaml` | Crossplane `Provider` for `upbound/provider-aws-s3` |
| `aws-s3-provider-runtime-config.yaml` | `DeploymentRuntimeConfig` with the S3 provider's resources |
| `default-provider-runtime-config.yaml` | `default` `DeploymentRuntimeConfig` with default provider resources |
| `aws-s3-provider-config.yaml` | `ProviderConfig` `aws-s3` that assumes the stage's IAM role |
| `aws-external-secret.yaml` | `ExternalSecret` `aws` with the AWS credentials of the stage |

All templates except the namespace are guarded by `.Capabilities.APIVersions.Has`. They are rendered only when the
cluster (or `helm template -a`) reports the API version of the custom resource. This prevents Argo CD sync errors
while a CRD is not ready yet.

`values.yaml` sets these top-level keys:

- `global.stage`: stage name (`local`), used in the secret key `/<stage>/crossplane/aws`.
- `aws.s3.roleARN`: IAM role the S3 `ProviderConfig` assumes.
- `crossplane`: overrides for the upstream chart (memory request of the Crossplane container).
- `providers.awsS3` and `providers.default`: image of the S3 provider and resources of both runtime configs.

### Secrets

We use [External Secrets](https://external-secrets.io) to manage the secrets needed for this deployment. The
`ExternalSecret` reads the key `/<stage>/crossplane/aws` from the `ClusterSecretStore` `awssm-parameter-store`. For
how to provide these secrets, see the
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
| `values-subchart-overrides.yaml` | Overrides for subchart values (see [Testing](#testing)) |
| `tests/` | helm-unittest suites; snapshots in `tests/__snapshot__/` are gitignored |
| `renovate.json` | Renovate configuration |
| `.github/workflows/` | CI workflows |

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
`.Capabilities.APIVersions.Has`; helm templating works offline, so they must be passed explicitly.

In the workbench:

```sh
 helm template crossplane . \
   -a aws.upbound.io/v1beta1 \
   -a external-secrets.io/v1/ExternalSecret \
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
adjust the output directory.

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
