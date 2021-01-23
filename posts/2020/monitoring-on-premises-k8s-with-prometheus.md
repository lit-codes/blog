---
title: "Monitoring on-premises kubernetes with Prometheus and Grafana"
date: 2020-12-14
tags: [DevOps,Monitoring]
---

## Architecture

In LitCodes have decided to setup our monitoring in the staging environment
using Prometheus and Grafana. Prometheus and Grafana go well together and they
are widely used as open-source solutions for variety of businesses. We chose
those because of the level of control we get from them.

In this article we will see how we used Prometheus and Grafana together to
monitor our hybrid cloud environment.

We will also see a few challenges when we used these tools on top of Kubernetes.

Our stack looks like this.

```
==== applications =====       ====== monitoring ======
|                     |       |                      |
| Nodejs + PostgreSQL |       | Grafana + Prometheus |
|                     |       |                      |
=======================       ========================
          |                              |
====== system =========                  |
|                     |                  |
|   Kubernetes (k3s)  |  ----------------^
|                     |
=======================
```

Each of the boxes above represent a separate namespace in our Kubernetes
cluster. The reason to choose a separate namespace for our monitoring tools
from our applications is that we want to make sure we have an isolated
environment for our running applications, any data breach in the monitoring
side should have minimal effect on our application side.

We also decided to run the monitoring tools on a separate node, mostly because
our staging environment consists of small computers (A blend of old laptops and
raspberry pi machines), and we needed more power for our monitoring dashboard
to work. In fact, our monitoring tools are going to be similar to what we will
have on our production servers. We use AWS Lightsail instances to deploy our
production cluster.

In the future we might want to migrate to EKS (once we hit more traffic than we
can handle with our current setup), and we expect minimal to zero extra work to
migrate our environment, including monitoring to that setup.

## Installation

To install the monitoring tools we use the
[prometheus-operator](https://github.com/helm/charts/tree/master/stable/prometheus-operator)
Helm chart.

We use the following configuration to set up Prometheus on our environments:

```yaml:title=prometheus-operator.yaml
namespaceOverride: "monitoring"
kube-state-metrics:
  nodeSelector:
    role: master-node
prometheusOperator:
  nodeSelector:
    role: prometheus
  admissionWebhooks:
    patch:
      nodeSelector:
        role: prometheus
prometheus:
  prometheusSpec:
    nodeSelector:
      role: prometheus
    scrapeInterval: 5m
    evaluationInterval: 5m
alertmanager:
  alertmanagerSpec:
    nodeSelector:
      role: alert-manager
nodeExporter:
  serviceMonitor:
    scrapeTimeout: 1m
grafana:
  nodeSelector:
    role: grafana
```

The above setup lets us run `Grafana` and `Prometheus` on arbitrary nodes, we
use a custom-made K8s label called `role` to identify each node of our
Kubernetes cluster. In reality these may all point to the same nodes, or
separate nodes may be used for monitoring (depending on the staging or
production environments and the workload).

We keep our Grafana secret in a separate file and add it to our `gitignore`.

```yaml:title=grafana-secret.yaml
grafana:
  adminPassword: [our admin password]
```

For security reasons we keep the actual password only in our password manager,
and share it with the team.

In the future we are planning to use Vault + Consul to keep our configurations
and secrets, but depending on the size of the team and number of secrets we are
going to share we might decide that the existing approach is good enough.

We then use the following command to run the monitoring tools:

```bash
helm repo update
helm search repo prometheus
helm install --create-namespace --namespace monitoring prometheus-opera \
    stable/prometheus-operator -f prometheus-operator.yaml -f grafana-secret.yaml
```

We can then check the created pods using the following command:

```bash
kubectl -n monitoring get pods -o wide
```

Also to see the Kubernetes services we use the following command:

```bash
kubectl -n monitoring get services -o wide
```

For local development we use port-forwarding to expose the Grafana and Prometheus editor web services:

```bash
GRAFANA_PORT=3000
PROMETHEUS_PORT=9090
GRAFANA=$(kubectl -n monitoring get pod -l app.kubernetes.io/name=grafana -o name)
PROMETHEUS=$(kubectl -n monitoring get pod -l app=prometheus -o name)
kubectl -n monitoring port-forward $GRAFANA $GRAFANA_PORT &
kubectl -n monitoring port-forward $PROMETHEUS $PROMETHEUS_PORT &
```

We use the following Traefik configuration to provide access to Grafana and
Prometheus editors. Prometheus editor endpoint is not authenticated, therefore
we currently use basic authentication on top of Traefik for that page.
