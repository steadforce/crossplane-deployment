# crossplane-deployment
deploy crossplane with argo-cd

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
  -a pkg.crossplane.io/v1 \
  -f values-subchart-overrides.yaml \
  -f values-local.yaml \
  --output-dir _local/local . 
```

You can use this command to check if the output is as you expect. The `-a` parameters are needed since we use the
helm feature `.Capabilities.APIVersions.Has` to determine if a `CR` is installable in the cluster or not. Since
helm templating is designed to work offline we have to list the supported `CR`. Using `.Capabilities.APIVersions.Has`
feature in templating prevents sync errors in argo-cd if a `CR` can't be applied since its `CRD` isn't ready.

## seal aws secrets for different environments

### local Environment

```
 # encrypt aws credentials
  cat <<EOF | kubeseal --cert ../argocd-bootstrap/secrets/sf-k8s-local/sealed-rsa-8192.pub --raw --from-file=/dev/stdin --namespace crossplane-system --name aws && echo
[default]
aws_access_key_id = <aws access key from vault>
aws_secret_access_key = <aws secret access key from vault>
EOF
```

### Dev Environment

```
 # encrypt aws credentials
  cat <<EOF | kubeseal --cert ../argocd-bootstrap/secrets/sf-k8s-dev/sealed-rsa-8192.pub --raw --from-file=/dev/stdin --namespace crossplane-system --name aws && echo
[default]
aws_access_key_id = <aws access key from vault>
aws_secret_access_key = <aws secret access key from vault>
EOF
```

### Prod Environment

```
 # encrypt aws credentials
  cat <<EOF | kubeseal --cert ../argocd-bootstrap/secrets/sf-k8s-prod/sealed-rsa-8192.pub --raw --from-file=/dev/stdin --namespace crossplane-system --name aws && echo
[default]
aws_access_key_id = <aws access key from vault>
aws_secret_access_key = <aws secret access key from vault>
EOF
```

### Put encrypted secret values into  values-<stage>.yaml

Add or update the values-<stage>.yaml with following code replacing the <> with corresponding values sealed above.

```
aws:
  sealedCredentials: <sealed aws credentials>
```
