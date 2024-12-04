# openshift-eck

### References

- [ECK Operator Docs](https://www.elastic.co/downloads/elastic-cloud-kubernetes)

### Install ECK Operator

This YAML defines a `Subscription` for the OpenShift Operator Lifecycle Manager (OLM), requesting the installation of the `elasticsearch-eck-operator-certified` operator from the `certified-operators` catalog source located in the `openshift-marketplace` namespace. It will automatically install updates from the `stable` channel. The subscription is named `elastic-cloud-eck` and resides in the `openshift-operators` namespace. 

```
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: elastic-cloud-eck
  namespace: openshift-operators
spec:
  channel: stable
  installPlanApproval: Automatic
  name: elasticsearch-eck-operator-certified
  source: certified-operators
  sourceNamespace: openshift-marketplace
```

### Create Namespace

```
apiVersion: project.openshift.io/v1
kind: Project
metadata:
  name: elastic-monitoring
  labels:
    app: elastic-monitoring
```

### Create Service Accounts

The first ServiceAccount, named metricbeat, is created in the elastic-monitoring namespace. The second ServiceAccount, named filebeat, is also created in the elastic-monitoring namespace. Both service accounts provide identities for processes that will run in pods within the elastic-monitoring namespace, related to monitoring metrics and logging, respectively.

```
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: metricbeat
  namespace: elastic-monitoring
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: filebeat
  namespace: elastic-monitoring
```
