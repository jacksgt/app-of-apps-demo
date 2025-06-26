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
