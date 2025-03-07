# Create HelmRepository and HelmRelease using helm template approach for Flux

Both the ```helm-repository.yaml``` and ```helm-release.yaml``` files share common patterns and values that can be templated to reduce repetition and make it easier to deploy multiple applications with Flux and Helm. By creating a reusable template, you can parameterize fields like the chart name, version, namespace, and repository URL, then generate these files for different applications (e.g., ```podinfo```, ```crossplane```, etc.) with minimal effort.

Since we have been working with Flux and Helm on kubernetes, let's explore this approach.


## Common Values to Template

From my previous examples (```podinfo```, ```crossplane```), the common elements include:
* HelmRepository:

    * ```metadata.name```: The repository name (e.g., ```podinfo```, ```crossplane```).
    * ```spec.url```: The chart repository URL.
    * ```spec.interval```: Sync interval (e.g., 1h).
    * ```metadata.namespace```: Typically ```flux-system```.
* HelmRelease:
    * ```metadata.name```: Matches the app name.
    * ```spec.chart.spec.chart```: The chart name.
    * ```spec.chart.spec.version```: The chart version or range.
    * ```spec.chart.spec.sourceRef.name```: References the HelmRepository name.
    * ```spec.interval```: Reconciliation interval (e.g., 5m).
    * ```metadata.namespace``` and ```spec.targetNamespace```: The target namespace for deployment.
    * ```spec.values```: Custom values (app-specific).

## Helm Chart Template

Since we are already using Helm with Flux, we can create a Helm chart to template these Flux resources. This leverages Helm’s templating engine.

## 1. Create a Helm Chart:

```shell
helm create flux-template
rm -rf flux-template/templates/*
```

## 2. Create appName<appName>-values.yaml:

Create `crossplane-values.yaml` file:

```shell
mkdir flux-template/crossplane
vim flux-template/crossplane/crossplane-values.yaml
```

*Take a look at `template-values.yaml` file in `flux-templates` folder, you will find the required variables and the customValues in case you want to add a customValues map.*


Add the common values:

```yaml
appName: crossplane
repoUrl: https://charts.crossplane.io/stable
chartVersion: "1.19.0"
namespace: crossplane-system
customValues:
  args:
    - "--debug"
  resources:
    requests:
      cpu: "100m"
      memory: "128Mi"
    limits:
      cpu: "200m"
      memory: "256Mi"
```

## 3. Add `helm-repository.yaml` Template:

Create `flux-template/templates/helm-repository.yaml`:

```yaml
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: HelmRepository
metadata:
  name: {{ .Values.appName }}
  namespace: flux-system
spec:
  interval: 1h
  url: {{ .Values.repoUrl }}
```

## 4. Add `helm-release.yaml` Template:

Create `flux-template/templates/helm-release.yaml`:

```yaml
# This is a HelmRelease CRD that Flux will use to deploy {{ .Values.appName }}.
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: {{ .Values.appName }}
  namespace: {{ .Values.namespace }}
spec:
  interval: {{ .Values.interval }}
  chart:
    spec:
      chart: {{ .Values.appName }}
      version: {{ quote .Values.chartVersion }}
      sourceRef:
        kind: HelmRepository
        name: {{ .Values.appName }}
        namespace: flux-system
  releaseName: {{ .Values.appName }}
  targetNamespace: {{ .Values.namespace }}
  install:
    createNamespace: true
  {{- if .Values.customValues }}
  # Optional customizations
  values:
    {{- range $key, $value := .Values.customValues }}
    {{ $key }}:
      {{- if kindIs "slice" $value }}
      {{- range $item := $value }}
      - {{ quote $item }}
      {{- end }}
      {{- else }}
      {{- toYaml $value | trim | nindent 6 }}
      {{- end }}
    {{- end }}
  {{- end }}
```
## 5. Generate Files:

For Crossplane:

```shell
helm template flux-template -f crossplane/crossplane-values.yaml --output-dir flux-template/crossplane/
```


## 6. Commit to Git:

```shell
git add crossplane/flux-template/templates
git commit -m "Add templated deployments"
git push
```

# Pros and Cons
`Pros`: Leverages Helm’s powerful templating, reusable across projects.

`Cons`: Requires Helm installed locally, slightly more setup.

Happy templating 😊
