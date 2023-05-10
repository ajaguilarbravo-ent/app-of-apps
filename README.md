# apps-of-apps

This repository contains a helm chart, which configures applications in Argo CD in a declarative way.

When rendered (e.g. by `helm template`), this helm chart renders a collection of Application and AppProject resources.

To keep the list of applications manageable, the configuration is split up into multiple values files: `values.yaml` (main/root installation) and additional `values-*.yaml` files... The main/root `values.yaml` file declares additional applications from the same `app-of-apps` chart, but with the custom `values-*.yaml` which defines applications for the given domain.

## example of separate app-of-apps apps

Add the configuration for the Argo CD apps into a new values file, e.g. `values-team-x.yaml`:

```yaml
# override project list to be empty, to not inherit them from values.yaml
# or you can also define/move some project to be owned by this values file
projects:
  - project-name

clusters:
  - server:      https://
    name:        
    # ...

    genericApps:
      - chart: 
        name: 
        namespace: 
        valueFiles:
          - values.yaml
      # ...
```

Define a new argocd-apps app in the main `values.yaml` file which uses the values file above:

```yaml
clusters:
  # ...
  - server:      https://k8s.cluster.local
    name:        k8s
    # ...

    genericApps:
      - chart: app-of-apps
        releaseName: 
        name: app-of-apps
        repoURL: https://
        namespace: argocd
        valueFiles:
          - values.yaml
```

## renaming an app in argocd

If the application changes in argocd (but not the helm release name), it is relatively easy to rename the app:

- create the new app in argocd without syncing it yet
- the new app should display warnings that the resources defined by it are already owned by another app
- sync the new app
- the old app should now display the warnings about the resources owned by another app (the newly synced app)
- delete the old app (choose non-cascading propagation policy: only the app will be removed from argocd, the resources will stay)

## initial setup

To install the root `app-of-apps` app, use this:

```bash
cat <<EOF | kubectl --namespace argocd apply -f -
---
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: app-of-apps
  namespace: argocd
  labels:
    chart: app-of-apps
spec:
  destination:
    namespace: argocd
    server: https://kubernetes.default.svc
  project: meta
  source:
    chart: app-of-apps
    repoURL: https://chart.20.103.115.121.nip.io/app-of-apps
    targetRevision: '>0.0.0'
EOF
```
