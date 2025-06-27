GitOps Repository Structure Overview

This document outlines a repository structure for managing Kubernetes applications using a GitOps workflow with ArgoCD. The structure is designed to separate configuration from application code, and environment-specific settings from common application definitions.

Directory Layout:

```yaml
.
├── k8s-apps
│   ├── dev
│   │   ├── app.yaml
│   │   └── httpd.yaml
│   ├── staging
│   │   ├── app.yaml
│   │   └── httpd.yaml
│   └── production
│       ├── app.yaml
│       └── httpd.yaml
├── charts
│   ├── app
│   │   ├── Chart.yaml
│   │   ├── values.yaml
│   │   └── templates
│   │       ├── _helpers.tpl
│   │       ├── deployment.yaml
│   │       └── service.yaml
│   └── httpd
│       ├── Chart.yaml
│       ├── values.yaml
│       └── templates
│           ├── _helpers.tpl
│           ├── deployment.yaml
│           └── service.yaml
└── src
    ├── app
    │   └── ... (Application source code)
    └── httpd
        └── ... (Application source code)
```


Explanation

* `k8s-apps/`: This is the root directory that ArgoCD will monitor. It contains environment-specific subdirectories.
  * `dev/`, `staging/`, `production/`: Each directory represents a deployment environment. It contains ArgoCD Application resource manifests. These manifests tell ArgoCD what to deploy (which application) and where to deploy it from (which source Git repository and chart).
  * `app.yaml`, `httpd.yaml`: These are the ArgoCD Application manifests. For example, k8s-apps/dev/httpd.yaml will define the httpd application for the dev environment, pointing to the charts/httpd Helm chart in this same repository.
* `charts/`: This directory contains the reusable Helm charts for your applications.
  * `app/`, `httpd/`: Each subdirectory is a self-contained Helm chart. This is the "source of truth" for the Kubernetes resources (Deployment, Service, Ingress, etc.) that make up your application.
  * `Chart.yaml`: Contains metadata about the chart (name, version, etc.).
  * `values.yaml`: Contains the default configuration values for the chart. These can be overridden by the ArgoCD Application manifest for each environment.
  * `templates/`: Contains the Kubernetes manifest templates, which are rendered by Helm using the values from values.yaml.
* `src/`: This directory contains the actual source code for your applications. For the purpose of this GitOps setup, this directory is not directly used by ArgoCD but is included for completeness to show where the application code would live.

This structure allows you to manage your application deployments across multiple environments in a clear, organized, and version-controlled way. Changes to application manifests or configurations are made through Git commits, which ArgoCD then automatically syncs to the OpenShift clusters.


## ArgoCD onboarding for applications

To onboard this repository to our Argo CD instance, we start off by creating the relevant namespaces.
It is important that these have the `argocd.argoproj.io/managed-by=openshift-gitops` label.
This label allows the Argo CD instance in the `openshift-gitops` namespace to manage resource in our application namespace.
Below we create three namespaces: one for the development environment, one for the staging environment and one for the production environment.
It is recommended to always split separate environments into individual projects (namespaces).

This step is to be done by the infrastructure team and should be managed via the global GitOps repository.

```yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: fof-jack-demo-dev
  labels:
    argocd.argoproj.io/managed-by: openshift-gitops
spec: {}
---
apiVersion: v1
kind: Namespace
metadata:
  name: fof-jack-demo-staging
  labels:
    argocd.argoproj.io/managed-by: openshift-gitops
spec: {}
---
apiVersion: v1
kind: Namespace
metadata:
  name: fof-jack-demo-production
  labels:
    argocd.argoproj.io/managed-by: openshift-gitops
spec: {}
```

as well as the Argo CD AppProject:

```yaml
---
kind: AppProject
apiVersion: argoproj.io/v1alpha1
metadata:
    name: fof-jack-demo
    namespace: openshift-gitops
    # Finalizer that ensures that project is not deleted until it is not referenced by any application
    finalizers:
      - resources-finalizer.argocd.argoproj.io
spec:
    sourceRepos:
    - '*'
    # Only permit applications to deploy to the specified namespace in the same cluster
    destinations:
    - namespace: fof-jack-demo-dev
      server: https://kubernetes.default.svc
    - namespace: fof-jack-demo-staging
      server: https://kubernetes.default.svc
    - namespace: fof-jack-demo-production
      server: https://kubernetes.default.svc
    # Allow all namespaced-scoped resources to be created, except for ResourceQuota, LimitRange, NetworkPolicy
    namespaceResourceBlacklist:
    - group: ''
      kind: ResourceQuota
    - group: ''
      kind: LimitRange
    - group: ''
      kind: NetworkPolicy
    sourceNamespaces:
    - fof-jack-demo-dev
    - fof-jack-demo-staging
    - fof-jack-demo-production
    roles:
    - description: admin access to project
      groups:
      - MY_ADMIN_GROUP
      name: admin
      policies:
      - g, MY_ADMIN_GROUP, admin
```



Now, the developers can deploy their "App-of-Apps" applications that will act as the "seed" for each environment:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: app-of-apps
  namespace: fof-jack-demo-dev
spec:
  project: fof-jack-demo
  source:
    repoURL: 'https://github.com/jacksgt/app-of-apps-demo.git'
    path: k8s-apps/dev
    targetRevision: HEAD
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: fof-jack-demo-dev
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
---
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: fof-jack-demo-staging
  namespace: openshift-gitops
spec:
  project: fof-jack-demo
  source:
    repoURL: 'https://github.com/jacksgt/app-of-apps-demo.git'
    path: k8s-apps/staging
    targetRevision: HEAD
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: openshift-gitops
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
---
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: fof-jack-demo-staging
  namespace: openshift-gitops
spec:
  project: fof-jack-demo
  source:
    repoURL: 'https://github.com/jacksgt/app-of-apps-demo.git'
    path: k8s-apps/production
    targetRevision: HEAD
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: openshift-gitops
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

## Resources

Additional materials that are helpful to study:

* https://github.com/redhat-developer/openshift-gitops-getting-started
* https://advanceddevsecopsworkshop.github.io/workshop/modules/main/03-advanced-gitops.html
* https://developers.redhat.com/blog/2025/03/05/openshift-gitops-recommended-practices
