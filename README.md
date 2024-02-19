# d1-infra

This repository is managed by the platform team who are responsible for
the Kubernetes infrastructure. 

## Scope and Access Control

This repository is used to define the Kubernetes infrastructure components such as:

- Cluster add-ons (CRD controllers, admission controllers, monitoring, logging, etc.)
- Cluster-wide definitions (Namespaces, Ingress classes, Storage classes, etc.)
- Pod security standards
- Network policies

This repository is reconciled on the cluster fleet by Flux as the **cluster admin**.
Access to this repository is restricted to the platform team and the
[Flux bot account](https://github.com/controlplaneio-fluxcd/d1-fleet?tab=readme-ov-file#create-a-github-account-for-flux).

## Repository Structure

This repository contains the following top directories:

- **components** dir contains Flux HelmReleases for cluster addons with custom configuration per environment
- **deploy** dir contains the Flux configuration referred from the `d1-fleet` repo for reconciling the components in a specific order
- **automation** dir contains the Flux configuration for automating the OCI chart updates of the Helm releases

A cluster component is defined in a directory with the following structure:

```sh
component/
├── controllers # CRD definitions and controllers
│   ├── base # common definitions (Namespaces, RBAC, HelmRepositories, HelmReleases)
│   ├── production # production specific HelmRelease values
│   └── staging # staging specific HelmRelease values
└── configs # Custom Resources of controllers
    ├── base # common definitions
    ├── production # production specific values
    └── staging # staging specific values
```

## Deployment Strategy

Changes to the `main` branch are automatically reconciled by Flux on the staging cluster.

```mermaid
sequenceDiagram
    actor me
    participant git as Git<br><br>repository
    participant sc as Flux<br><br>source-controller
    participant kc as Flux<br><br>kustomize-controller
    participant kube as Kubernetes<br><br>api-server
    participant nc as Flux<br><br>notification-controller
    me->>git: 1. git push
    sc->>git: 2. git pull
    sc->>sc: 3. build artifact for revision
    sc->>kube: 4. update status for revision
    sc-->>nc: 5. emit events
    kube->>kc: 6. notify about new revision
    kc->>sc: 7. fetch artifact for revision
    kc->>kc: 8. build manifests to objects
    kc->>kube: 9. reconcile objects
    kc-->>kube: 10. wait for readiness
    kc->>kube: 11. update status for revision
    kc-->>nc: 12. emit events
    nc-->>me: 13. send alerts for revision
```

After the changes are validated, the platform team can promote the changes to the production clusters
by merging the `main` branch into the `production` branch.

## Helm Release Automation

The staging cluster runs the Flux image automation controllers which automatically
update the HelmRelease definitions with the latest chart version.

When a new chart version is pushed to the container registry, and if it matches the semver policy,
Flux will update the HelmRelease YAML definitions and will push the changes to the `main` branch.

```mermaid
sequenceDiagram
    actor me
    participant oci as OCI<br><br>repository
    participant irc as Flux<br><br>image-reflector-controller
    participant iac as Flux<br><br>image-automation-controller
    participant kube as Kubernetes<br><br>api-server
    participant git as Git<br><br>repository
    me->>oci: 1. helm push oci://chart:version
    irc->>oci: 2. list versions
    irc->>irc: 3. match versions to policy
    irc->>kube: 4. update status
    kube->>iac: 5. notify about new version
    iac->>git: 6. git checkout main
    iac->>iac: 7. patch HelmRelease with new chart version
    iac->>git: 8. git push origin main
    iac->>kube: 9. update status
```

After the changes are pushed to the `main` branch, the HelmReleases will be upgraded to the new
chart versions on the staging cluster.

```mermaid
sequenceDiagram
    participant git as OCI<br><br>repository
    participant sc as Flux<br><br>source-controller
    participant hc as Flux<br><br>helm-controller
    participant kube as Kubernetes<br><br>api-server
    sc->>git: 1. helm pull chart
    sc->>sc: 2. build chart for revision
    sc->>kube: 3. update chart status
    kube->>hc: 4. notify about new revision
    hc->>sc: 5. fetch chart
    hc->>kube: 6. get release values
    hc->>hc: 7. upgrade release
    hc-->>kube: 8. run tests
    hc-->>kube: 9. wait for readiness
    hc->>kube: 10. update status
```
