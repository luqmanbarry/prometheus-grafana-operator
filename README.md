# Prometheus Grafana Operator

## Overview

A helm chart for installing the grafana-operator, adding a datasource, setting up a grafana instance, dashboards and alerts. This chart was tested against an RHOCP cluster. However, you may augment it and make it installable onto other Kubernetes flavors.

Installing this chart will setup the following resources:

1. Admin level resources
- OperatorGroup
- CatalogSource
  - represents a store of metadata that OLM can query to discover and install operators and their dependencies.
  - https://olm.operatorframework.io/docs/concepts/crds/catalogsource/[Click here for more]
- ServiceAccount
  - its purpose is to authenticate against the metrics server  via a `Bearer SA_TOKEN`

2. Non Admin level resources
- Subscription
  - represents an intention to install an operator. It is the CustomResource that relate an operator to a CatalogSource.
  -  https://olm.operatorframework.io/docs/concepts/crds/subscription/[Click here for more]
- ConfigMap
  - used for mounting certificates
- Secret
  - for securing the session cookie
- Grafana Operator CRDs
  - GrafanaDataSource
  - Grafana
  - GrafanaDashboard
  - GrafanaNotifictionChannel
  - these are CRDs that comes with installing the Grafana Operator.
  - [Click here for more](https://github.com/grafana-operator/grafana-operator)

## Installation Guide

Follow these steps to setup the monitoring stack and related dashboards.

### Prerequisites

- User Workload monitoring is enabled
  - [Follow the steps in `Enabling User Workload Monitoring`](https://docs.openshift.com/container-platform/4.9/monitoring/enabling-monitoring-for-user-defined-projects.html)
  - Ensure you have access to an OpenShift or Kubernetes cluster.
  - Validate you have enough privileges to deploy OperatorGroup and CatalogSource CRDs.
  - for this, a user may rely on Infra team for a ServiceAccount or have someone else with elevated privileges apply them.
- `Helm`, `oc/kubectl`, `git(optional)` installed on workstation from which commands are run.
- WORKDIR is assumed to be the root directory of the parent helm chart.

Feel free to use `helm template ... | oc apply -f -` or `helm upgrade --install ...`. For this guide I will be utilizing the `helm upgrade --install` command.

### Procedure

1. Install the olm-resources subchart

This step requires elevated privileges. In a multi-tenant environment there might be a Infra/DevOps team handling this responsibility.

- Verify chart is syntactically correct.

```
# COMMAND
helm lint ./charts/olm-resources

# OUTPUT
==> Linting ./charts/olm-resources
[INFO] Chart.yaml: icon is recommended

1 chart(s) linted, 0 chart(s) failed
```

- Install the olm-resources subchart.

  - Verify Chart

```
# Fill in NAMESPACE
helm upgrade --install olm-resources ./charts/olm-resources -n  NAMESPACE --dry-run
```
- Install Chart Post Verification
```
# Fill in NAMESPACE
helm install --upgrade olm-resources ./charts/olm-resources -n NAMESPACE
```
- Expected Output
```
# OUTPUT
Release "olm-resources" does not exist. Installing it now.
NAME: olm-resources
LAST DEPLOYED: Wed Feb  2 09:29:34 2022
NAMESPACE: monitoring
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Installs the olm-resources.
```

2. Install the grafana-resources CRDs.

**Prerequisites**:

 Update the following values in `WORKDIR/charts/grafana-resources/values.yaml`

```
selectors:
  grafanaOperator:
    selectionKey: app-team
    selectionValue: abc-acb
  prometheusQueries:
    # Replace with your namespace names regex patterns
    namespace:
      whitelist: "abc-.*|acb-.*"
      blacklist: ".*-bld"
    # Replace with your pod names regex patterns
    pod:
      whitelist: "web.*|.*service.*"
      blacklist: ".*-build"   
```

---

a. Verify chart is syntactically correct.

```
# COMMAND
helm lint ./charts/grafana-resources

# OUTPUT
==> Linting grafana-resources
[INFO] Chart.yaml: icon is recommended

1 chart(s) linted, 0 chart(s) failed
```

b. Install the grafana-resources.

The serice account(grafana-thanos) with *cluster-monitoring-view* role will be provided by infra/DevOps team; seek their assistance before proceeding.

You will need to reference your own ServiceAccount if olm-resources subchart was not installed.

- Verify the chart

```
# Fill in SERVICE_ACCOUNT_NAME, NAMESPACE

# Verify chart is installable
helm upgrade --install grafana-resources ./charts/grafana-resources --set grafanaDataSource.auth.bearerToken="$(oc sa get-token SERVICE_ACCOUNT_NAME -n NAMESPACE)" --set grafanaInstance.serverRootUrl="$(oc get route grafana-route -o jsonpath={.spec.host} -n NAMESPACE)" -n NAMESPACE --dry-run
```

- Install the chart post verification

```
# COMMAND
helm upgrade --install grafana-resources ./charts/grafana-resources --set grafanaDataSource.auth.bearerToken="$(oc sa get-token grafana-thanos -n abc-monitoring)" --set grafanaInstance.serverRootUrl="$(oc get route grafana-route -o jsonpath={.spec.host} -n abc-monitoring)" -n abc-monitoring
```

- Expected Output
```
# OUTPUT
elease "grafana-resources" has been upgraded. Happy Helming!
NAME: grafana-resources
LAST DEPLOYED: Thu Feb 17 17:39:58 2022
NAMESPACE: abc-monitoring
STATUS: deployed
REVISION: 55
TEST SUITE: None
NOTES:
Installs Grafana resources.

This include the following:
  - Grafana
  - GrafanaDataSource
  - GrafanaDashboard
  - GrafanaNotificationChannel
  - PrometheusRule: Set alerts.prometheusRules.enabled to true for installation.
  - AlertManager: Set alerts.alertManager.enabled to true for installation.
    Configuration: Incomplete, provide missing configs.
```

If you get a *"grafana-route" not found* error, just reapply the chart, it will find the route the second time.


### Installation Verification

Post installation of the charts above steps, you should see the following resources:

- Pods running in the monitoring namespace

```
# Fill in NAMESPACE
oc get pods -n NAMESPACE

# OUTPUT
NAME                                                   READY   STATUS    RESTARTS   AGE
grafana-deployment-59c7bf4d7f-stzd5                    2/2     Running   0          7d23h
grafana-operator-controller-manager-676bbd6cf9-jqg44   2/2     Running   0          26d
```

- Service Instances
```
# Fill in NAMESPACE
oc get service -n NAMESPACE

# OUTPUT
NAME                                                  TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)             AGE
grafana-operator-controller-manager-metrics-service   ClusterIP   xxx.xxx.xxx.xxx    <none>        8443/TCP            26d
grafana-service                                       ClusterIP   xxx.xxx.xxx.xxx   <none>        3000/TCP,9091/TCP   7d23h
```

- A route instance

```
# Fill in NAMESPACE
oc get route -n NAMESPACE

# OUTPUT
NAME            HOST/PORT                                                               PATH   SERVICES          PORT            TERMINATION   WILDCARD
grafana-route   grafana-route-NAMESPACE.apps.MY-DOMAIN.com   /      grafana-service   grafana-proxy   reencrypt     None
```

Test accessing the grafana web ui 

- Grab the route host

```
# Fill in NAMESPACE
oc get route grafana-route -o jsonpath={.spec.host} -n NAMESPACE

# OUTPUT
grafana-route-NAMESPACE.apps.MY-DOMAIN.com
```
- Open a browser and goto the url captured in above step
  - Sign in with your OpenShift credentials.

