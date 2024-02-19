# d1-infra

This repository is managed by the platform team who are responsible for
the Kubernetes infrastructure. 

## Scope and Access Control

This repository is used to define the Kubernetes infrastructure components such as:

- Cluster add-ons (CRD controllers, admission controllers, monitoring, logging, etc.)
- Cluster-wide definitions (Namespaces, Ingress classes, Storage classes, etc.)
- Pod security standards
- Network policies

This repository is reconciled on the cluster fleet by Flux under
the **cluster admin** Kubernetes cluster role. Access to this repository
is restricted to the platform team and the
[Flux bot account](https://github.com/controlplaneio-fluxcd/d1-fleet?tab=readme-ov-file#create-a-github-account-for-flux).
The main branch should be protected to require a pull request to merge changes.

