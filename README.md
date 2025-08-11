# crossplane-deployment
deploy crossplane with argo-cd

## secrets

We use [external secrets](https://external-secrets.io) to manage the secrets needed for this deployment.
For documentation how to provide these secrets take a look into
[external-secrets-deployment](https://github.com/steadforce/external-secrets-deployment) README.md.

## Testing

### values-subchart-overrides.yaml

The `values-subchart-overrides.yaml` file is used to override values in the subchart(s) used by this chart.
We have to separate the values for the subcharts from the values for the main chart, to be able to
unit test for incompatible changes in values of the subcharts. This is necessary because helm does not allow
switching off the usage of values.yaml. Now it's possible to test if we use the same registry and repository
for images as the subcharts are using.

### run helm unittests

```shell
 docker run --pull=always -ti --rm -v "$(pwd):/apps" -u $(id -u) helmunittest/helm-unittest .
```

Or with output in JUnit format:

```shell
 docker run --pull=always -ti --rm -v "$(pwd):/apps" -u $(id -u) helmunittest/helm-unittest -o test-output.xml .
```

### Render helm charts
The following command renders the charts like argo-cd does for local deployment:

```
 helm template --release-name crossplane -n crossplane-system --include-crds --skip-tests \
  -a aws.upbound.io/v1beta1 \
  -a external-secrets.io/v1beta1/ExternalSecret \
  -a pkg.crossplane.io/v1 \
  -a pkg.crossplane.io/v1beta1 \
  -f values-subchart-overrides.yaml \
  -f values-local.yaml \
  --output-dir _local/local . 
```

You can use this command to check if the output is as you expect. The `-a` parameters are needed since we use the
helm feature `.Capabilities.APIVersions.Has` to determine if a `CR` is installable in the cluster or not. Since
helm templating is designed to work offline we have to list the supported `CR`. Using `.Capabilities.APIVersions.Has`
feature in templating prevents sync errors in argo-cd if a `CR` can't be applied since its `CRD` isn't ready.

## Run GitHub pipeline locally

To run the GitHub pipeline in the local environment, start the workbench, cd into the folder containing this
`README.md` and execute the following command:

```shell
  act
```

On first execution you're asked which flavour of the act image should be used. Using the default `medium`
is a good starting point.
