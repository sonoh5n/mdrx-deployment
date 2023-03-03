# Architecture

Alex Dutton and Paul walk, 2021-02-25

![mdrx_deployment_overview](https://github.com/antleaf/mdr-x-deployment/blob/main/docs/mdrx_deployment_overview.png)

## Design overview

Almost everything is deployed using Ansible playbooks, with various components split out into separate Ansible roles. The playbooks support multiple deployment environments, each corresponding to a Google Cloud Platform project.
Environment-specific configuration is found in [vars/environment/](https://github.com/antleaf/mdr-x-deployment/tree/main/vars/environment), and defines the GCP project details, the top-level domain name, and the instances in that environment.

## Instances

An MDR-X instance is the pairing of the MDR-X application with a full deployment of GitLab. Off-the-shelf components are deployed and managed by Helm - this includes cert-manager, Prometheus, GitLab, and the MDR-X supporting PostgreSQL and Redis services. Each MDR-X instance is contained in its own
Kubernetes namespace with an `instance-` prefix (e.g. `instance-alpha`,`instance-beta`).

The MDR-X application is deployed via templated Kubernetes resource manifests.

The `mdrx_instance` role manages these instance GitLab/MDR-X pairings,
including:

* importing the `gitlab` and `mdrx_app` roles to deploy each application

* provisioning OAuth application credentials via a GitLab Rails console script

* maintaining an instance secret containing credentials shared across
  applications:

  * the MDR-X PostgreSQL and Redis passwords
  * the MDR-X Flask `SECRET_KEY`
  * OAuth credentials issued by GitLab to MDR-X


Some GitLab Helm features are turned off so that we can manage them directly
outside of the Helm chart. These are:

* Prometheus, as we deploy an environment-wide Prometheus instance via Helm
* cert-manager, as again, we deploy an environment-wide instance via Helm

## Project-wide services

Prometheus is installed in the `prometheus` namespace via the `prometheus`
Ansible role. It is currently unsecured.

Cert-manager is installed in the `cert-manager` namespace via the
`ingress_controller` role. This role also installs the nginx ingress controller
in the `default` namespace.
