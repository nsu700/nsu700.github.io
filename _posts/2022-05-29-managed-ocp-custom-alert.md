---
layout: post
title: Managed-OCP custom alert
date: 2022-05-29 19:52:24.000000000 +08:00
tags: [OCP, alert]
---

# General

This guide is heavily influenced by RH black belt teams’ [guide](https://mobb.ninja/docs/rosa/federated-metrics-prometheus/) for ROSA, but it not only works for ROSA

It will need prometheus and grafana operators for this implementation, if you dont need grafana, just ignore it

# Prepare Environment

1. Set the namespace to locate the custom prometheus and alertmanager

```go
export NAMESPACE=federated-metrics
```

1. Create the project

```go
oc new-project $NAMESPACE
```

1. Install operators in the OCP console
    1. Prometheus
        1. Operators ⇒ Search “Prometheus” in search bar ⇒ Click the `Prometheus Opertor` ⇒ Click `Continue` on the warning page ⇒ Click `Install` ⇒ In the `Installation mode`, choose `A specific namespace on the cluster` and pick the Project Name in the drop down list ⇒ Click `Install`
    2. Grafana (Optional)
        1. Operators ⇒ Search “Grafana” in search bar ⇒ Click the `Grafana Operator` ⇒ Click `Continue` on the warning page ⇒ Click `Install` ⇒ In the `Installation mode`, choose `A specific namespace on the cluster` and pick the Project Name in the drop down list ⇒ Click `Install`
        

# Deploying the monitoring stack

1. Verify the operators installed successfully in previous steps

```go
❯ oc get pods -n $NAMESPACE
NAME                                                   READY   STATUS    RESTARTS      AGE           
grafana-operator-controller-manager-7c84b74f89-776wm   2/2     Running   0             46m
prometheus-operator-86757886d8-gljh6                   1/1     Running   0             46m
```

1. Install the `mobb/rosa-federated-prometheus` helm chart

```go
❯  helm upgrade --install -n $NAMESPACE monitoring \
   --set grafana-cr.basicAuthPassword='mypassword' \
   --set fullnameOverride='monitoring' \
   --version 0.5.1 \
   mobb/rosa-federated-prometheus
Release "monitoring" does not exist. Installing it now.
```

1. Verify the `prometheus`, `alertmanager` are up and running

```go
❯ oc get pods
NAME                                                   READY   STATUS    RESTARTS      AGE
alertmanager-monitoring-alertmanager-cr-0              2/2     Running   0             36m
grafana-deployment-95d7dd7f6-6bbrq                     2/2     Running   0             70m
grafana-operator-controller-manager-7c84b74f89-776wm   2/2     Running   0             71m
prometheus-monitoring-prometheus-cr-0                  3/3     Running   1 (69m ago)   70m
prometheus-monitoring-prometheus-cr-1                  3/3     Running   1 (69m ago)   70m
prometheus-operator-86757886d8-gljh6                   1/1     Running   0             71m
```

# How to create slack webhook for notification

1. Open the page [slack webhook](https://api.slack.com/messaging/webhooks)
2. Click `Create your slack app`
3. Choose `From scratch`
4. Input your App name and select a workspace
5. Click `Incoming Webhooks`
6. Switch on the `Activate Incoming Webhooks`
7. Click `Add New Webhook to Workspace`
8. Select a channel in the drop-down list, Click Allow
9. The `Webhook URL` is the one we need in next steps

# Create custom alert-receiver

1. Create yaml file for alertmanager config

```go
❯ cat monitoring-alertmanager-cr-config.yaml
global:
  slack_api_url: $SLACK_API
receivers:
- name: slack-notifications
  slack_configs:
  - channel: '#alertmanager'
    send_resolved: true
route:
  group_by:
  - alertname
  receiver: slack-notifications
```

1. Create secret for alertmanager config

```go
❯ oc create secret generic custom-alert-manager-config --from-file=alertmanager.yaml=monitoring-alertmanager-cr-config.yaml -n $NAMESPACE
secret/custom-alert-manager-config created
```

1. Update the alertmanager to reference the custom config

```go
oc edit alertmanagers.monitoring.coreos.com monitoring-alertmanager-cr -n $NAMESPACE

apiVersion: monitoring.coreos.com/v1
kind: Alertmanager
metadata:
...
spec:
  configSecret: custom-alert-manager-config => change this to the secret name
  replicas: 1 => modify this to replicas of alert manager, by default it is 3
  securityContext: {}
```

1. Verify the alertmanager loaded the new config

```go
❯ oc debug alertmanager-monitoring-alertmanager-cr-0 -- cat /etc/alertmanager/config/alertmanager.yaml
Defaulting container name to alertmanager.
Use 'oc describe pod/alertmanager-monitoring-alertmanager-cr-0-debug -n federated-metrics' to see all of the containers in this pod.

Starting pod/alertmanager-monitoring-alertmanager-cr-0-debug ...
global:
  slack_api_url: $SLACK_API
receivers:
- name: slack-notifications
  slack_configs:
  - channel: '#alertmanager'
    send_resolved: true
route:
  group_by:
  - alertname
  receiver: slack-notifications

Removing debug pod ...
```

# Create custom prometheus rules

1. Create yaml file for alert rules, make sure the resource should be assigned two labels `prometheus: monitoring-prometheus-cr` and `role: alert-rules`

```go
❯ cat monitoring-custom-rules.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  labels:
    prometheus: monitoring-prometheus-cr
    role: alert-rules
  name: custom-rules
spec:
  groups:
  - name: Node
    rules:
    - alert: CPUUtilizationHigh
      annotations:
        description: 'CPU utilization high, curreny utility is  {{ printf "%.2f" $value }}%, on Node {{ $labels.node}}'
      expr: |
        100 * instance:node_cpu_utilisation:rate1m > 75
      for: 30m
      labels:
        ObjectId: '{{$labels.instance}}'
        cluster: testaro
        severity: critical
        testlabel: "true"
    - alert: MemUtilizationHigh
      annotations:
        description: 'RAM utilization high, curreny utility is  {{ printf "%.2f" $value }}%, on Node {{ $labels.node}}'
      expr: |
        instance:node_memory_utilisation:ratio * 100 > 75
      labels:
        ObjectId: '{{$labels.instance}}'
        cluster: testaro
        severity: critical
        testlabel: "true"
```

1. Create the prometheusrule

```go
❯ oc apply -f monitoring-custom-rules.yaml
prometheusrule.monitoring.coreos.com/custom-rules created
```

1. Verify the prometheusrules loaded into prometheus instance

```go
debug into the prometheus pod and `cat /etc/prometheus/rules/prometheus-monitoring-prometheus-cr-rulefiles-0/*yaml`
```

# Validate alerts

1. Get the prometheus route

```go
oc -n ${NAMESPACE} get route prometheus-route
```

1. Open the route and verify the alert page
2. Verify the slack channel