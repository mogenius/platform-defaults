# platform-defaults

The chart version and the values the mogenius operator installs each platform
component with. One file per component, no templating, no logic — the operator
fetches the file, takes `spec.version` and merges `spec.valuesObject` under
whatever the cluster's own `PlatformConfig` says.

Changing a file here changes what every cluster gets on its next reconcile.
There is no release step and no staging: `main` is live.

## How a file is used

The operator builds the URL itself:

```
<spec.platformSource>/<spec.platformVersion>/<component>.yaml
```

with `platformSource` defaulting to
`https://raw.githubusercontent.com/mogenius/platform-defaults/refs/heads` and
`platformVersion` to `main`. So the default resolves to this repository's `main`
branch, and a cluster can pin itself to a branch or a fork by setting those two
fields.

The `<component>` part is fixed — it is the operator's component name, which is
why the file names cannot be changed freely:

| File | Installed as |
| --- | --- |
| `argocd.yaml` | Argo CD, when `spec.gitOps.argocd.enabled` |
| `flux-operator.yaml` | Flux, when `spec.gitOps.fluxcd.enabled` |
| `cert-manager.yaml` | cert-manager |
| `traefik.yaml` | Traefik |
| `external-dns.yaml` | external-dns |
| `external-secrets-operator.yaml` | external-secrets |
| `kube-prometheus-stack.yaml` | kube-prometheus-stack |
| `loki.yaml` | Loki |
| `alloy.yaml` | Grafana Alloy |
| `renovate-operator.yaml` | mogenius renovate-operator |

## File format

```yaml
apiVersion: mogenius.com/v1alpha1
kind: PlatformDefault
spec:
  # renovate: datasource=helm registryUrl=<repo url> chart=<chart name>
  version: 1.2.3
  valuesObject:
    # helm values, exactly as the chart expects them
```

`valuesObject: {}` is valid and means "the chart's own defaults" — but only when
the chart actually installs without values. Loki does not, which is how it sat
uninstallable for months.

Values here are the base layer. Per cluster they are merged with, in order of
increasing precedence: values the operator derives from the `PlatformConfig`
(issuers, a `ServiceMonitor` when the CRD exists, …), then any `PlatformPatch`
the cluster references. So a cluster can override anything in these files
without touching this repository.

## Validating a change

Do this before merging. Renovate cannot, and a broken file is not visible until
a cluster tries to install it — the operator applies a `HelmRelease` that fails
later, inside the GitOps engine.

```bash
# once
helm repo add traefik https://helm.traefik.io/traefik && helm repo update

# for the file you touched: render the pinned chart with its values
helm template check traefik/traefik --version 41.5.0 \
  --kube-version 1.31.0 \
  -f <(yq '.spec.valuesObject' traefik.yaml)
```

`--kube-version` matters: without it helm assumes a very old cluster and several
charts refuse to render for reasons that have nothing to do with your change.

An empty output means the values satisfy the chart's `values.schema.json` and
its templates. That is the whole check, and it catches every failure found so
far.

## Two traps this repository has already fallen into

**A version bump does not carry values forward.** Renovate changes the
`version:` line and nothing else. Traefik went from 39.0.7 to 41.5.0, where
`logs.access.enabled` had been renamed to `accessLog.enabled`; the chart's
schema rejects unknown keys outright, so every install failed with
`additional properties 'logs' not allowed`. The operator reported the component
as ready regardless, because writing the `HelmRelease` had succeeded.

**A pin above every published version is invisible.** Loki was pinned to
`13.6.1` while the chart's newest release is in the 7.x line. Renovate sees
nothing newer and stays silent, so it looks maintained. Only rendering it shows
the truth.

Both mean the same thing in practice: when you touch a `version:` line, read the
chart's changelog for renamed values, and run the render above.

## Renovate

`renovate.json` carries two annotation forms, because the charts come from two
kinds of registry:

```yaml
# renovate: datasource=helm registryUrl=https://helm.traefik.io/traefik chart=traefik
# renovate: datasource=docker packageName=ghcr.io/controlplaneio-fluxcd/charts/flux-operator
```

The second is for charts published to an OCI registry — `flux-operator` is one,
and it previously used the `registryUrl` form pointing at a URL that answers
404, which silently froze its version.

If you add a component, check that Renovate resolves it: a wrong `registryUrl`
or `packageName` fails the same quiet way.
