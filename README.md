
## This is a highly experimental project to deploy the new CF Kubernetes-centric components on Kubernetes. It is **not** meant for use in production and is subject to change in the future.

## Please direct all questions to [#release-integration](https://cloudfoundry.slack.com/archives/C0FAEKGUQ) slack channel and ping @interrupt

---------

# Cloud Foundry for Kubernetes

## Table of Contents
* <a href='#purpose'>Purpose</a>
* <a href='#deploy-cf4k8s'>Deploying CF for K8s</a>
* <a href='#future'>What's next</a>

### <a name='purpose'></a> Purpose
Cloud Foundry for Kubernetes (CF4K8s) is a  canonical deployment artifact for deploying the Cloud Foundry Application Runtime on Kubernetes. 

#### Kubernetes native
CF4K8s is built from ground up to leverage Kubernetes native features 

#### Built on top of Kubernetes ecosystem projects
CF4K8s builts on top of well known enterprise ready projects like [Istio](https://github.com/istio/istio), [envoy](https://github.com/envoyproxy/envoy), [fluentd](https://www.fluentd.org/) and [kpack](https://github.com/pivotal/kpack)

### <a href='#deploy-cf4k8s'></a> Deploying CF for K8s

#### Prerequisites

You need the following CLIs on your system to be able to run the script:
* [`kapp`](https://k14s.io/#install)
* [`helm`](https://github.com/helm/helm#install)
* [`ytt`](https://k14s.io/#install)

In addition, you will also probably want [`kubectl`](https://kubernetes.io/docs/tasks/tools/install-kubectl/) for your own debugging and inspection of the system.

Make sure that your Kubernetes config (e.g, `~/.kube/config`) is pointing to the cluster you intend to
deploy CF for K8s to. This cluster should be on an IaaS that supports load
balancer services (e.g., GKE, AKS, etc.).

#### Steps to deploy

1. Git clone this repository and `cd` into this directory.
2. Update the submodules of this repository: `git submodule update --init --recursive`.
3. Deploy a database that can be used for the Cloud Controller's DB.
  - For example:
  ```
  helm repo add stable https://kubernetes-charts.storage.googleapis.com
  helm upgrade --install capi-database stable/postgresql -n default -f <(cat <<EOF
  initdbScripts:
    setup_db.sql: |
      CREATE DATABASE cloud_controller;
      CREATE ROLE cloud_controller LOGIN PASSWORD 'cloud_controller';
    hello_world.sh: |
      #!/bin/bash
      echo "hello, world!"
      psql -U postgres -f /docker-entrypoint-initdb.d/setup_db.sql
      psql -U postgres -d cloud_controller -c "CREATE EXTENSION citext"
      psql -U postgres -d cloud_controller -c "create sequence bobby"
  EOF
  )
  ```
4. Create a file called `cf-install-values.yml`. You can use `sample-cf-install-values.yml` in this directory as a starting point.
5. Change the `system_domain` and `app_domain` to your desired domain address 
6. Generate certificates for the above domains and paste them in `crt`, `key`, `ca` values
7. From this directory, run `bin/install-cf.sh <path to your cf-install-values.yml file>`
8. Configure DNS on your IaaS provider to point the wildcard subdomain of your
   system domain and the wildcard subdomain of all apps domains to point to external IP
   of the Istio Ingress Gateway service. You can retrieve the external IP of this service by running
   `kubectl get svc -n istio-system istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[*].ip}'`
9. Set up cf cli to point to CF using `cf api --skip-ssl-validation https://api.<your domain>` and then 
 auth by running `cf auth admin cfadminpassword`
10. Create orgs and spaces and enable docker feature by running `cf enable-feature-flag diego_docker`
11. Finally, run `cf push diego-docker-app -o cloudfoundry/diego-docker-app` and verify by visiting the URL

### <a href='#future'></a> What's next

Our plan is to release an alpha version of CF4K8s to the community in Feb 2020, which will include build packs based `cf push` experience.

The alpha version will enable the CF project teams to integrate and ship new capabilities for CF4K8s. In addition, we intend to provide a set of tests to validate features before shipping releases.
 
Next up, we plan to build continuous integration (CI) support - a set of CI tasks - which will enable teams to deploy their own pipeline to integrate other components, validate features and cut new releases (just like they do today in the CF4Bosh world). In addition, the release integration team plans to use the same CI tooling to build CF4K8s integration workflows to ship versioned CF4K8s artifacts.

Once we achieve the first two milestones, we intend to explore the CF user needs (platform engineers) to build an enterprise-ready CF4K8s artifact to deploy Cloud Foundry on K8s, with features that CF users are accustomed to today with cf-deployment.
