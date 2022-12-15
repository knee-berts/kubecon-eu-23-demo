# KubeCon EU 2023 Presentation

## Management GKE Cluster

The Management GKE cluster manages the lifecycle of Workload Clusters. A Management Cluster is also where one or more providers run, and where resources such as Cluster are deployed and stored.

There is a chicken and egg scenario where the Management GKE cluster cannot create itself; There are two options to initialize the Management GKE cluster:

### 1. (PREFERRED) Create the Management GKE cluster via KRM

To be able to create the Management GKE cluster via KRM there will need to be an existing cluster that can temporarily act as the Management Cluster; This can be a locally hosted KIND ("Kubernetes In Docker") instance from a Engineers machine.

In this demo the KRM solution will be Crossplane, however, other alternatives such as KCC (Kubernetes Config Connector) exist and are good options.

Note: Due to this cluster being an intermediate step to initialize the real Management GKE cluster the following steps will not be initially automated.

There are some assumptions to the following steps that should be recognized. Firstly, it's expected a GCP project already exists and a GSA has been created with adequate project level IAM to be able to create GCP resources through the interim locally hosted Management Kubernetes cluster via KRM resources.

1. Install Crossplane into temporary locally hosted Management Kubernetes cluster.

2. Deploy Crossplane configuration; EG. Provider, ProviderConfig

EG.

```yaml
---
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-gcp
  namespace: crossplane-system
spec:
  package: xpkg.upbound.io/upbound/provider-gcp:v0.18.0
---
# GCP ProviderConfig with service account secret reference
apiVersion: gcp.upbound.io/v1beta1
kind: ProviderConfig
metadata:
  name: default
  namespace: crossplane-system
spec:
  projectID: raspbernetes
  credentials:
    source: Secret
    secretRef:
      namespace: crossplane-system
      name: gcp-creds
      key: key
---
apiVersion: v1
kind: Secret
metadata:
    name: gcp-creds
    namespace: crossplane-system
data:
    key: REDACTED
```

3. Define and Deploy Management GKE Cluster via Crossplane KRM `Cluster` resource.

EG.

```yaml
---
# https://doc.crds.dev/github.com/crossplane/provider-gcp/container.gcp.crossplane.io/Cluster/v1beta2@v0.22.0
apiVersion: container.gcp.upbound.io/v1beta1
kind: Cluster
metadata:
  name: auto-pilot-gke-cluster-0
  namespace: crossplane-system
spec:
  forProvider:
    description: "GKE Auto Pilot Management Cluster"
    # TODO: Set an initial GKE Auto Pilot version explicitly
    # initialClusterVersion:
    releaseChannel:
      # Possible Values: "UNSPECIFIED", "RAPID", "REGULAR", "STABLE"
      channel: RAPID
    maintenancePolicy:
      window:
        recurringWindow:
          # make the window encompass every weekend from midnight Saturday till the last minute of Sunday UTC:
          recurrence: "start time = 2019-01-05T00:00:00Z end time = 2019-01-07T23:59:00Z recurrence = FREQ=WEEKLY;BYDAY=SA"
    location: us-central1
    # https://cloud.google.com/kubernetes-engine/docs/concepts/autopilot-overview
    enableAutopilot: true
    # https://cloud.google.com/kubernetes-engine/docs/how-to/confidential-gke-nodes
    confidentialNodes:
      enabled: false
    enableBinaryAuthorization: true
    authenticatorGroupsConfig:
      securityGroup: gke-security-groups@<DOMAIN>.com
    addonsConfig:
      configConnectorConfig:
        enabled: true
    # Possible values: "DATAPATH_PROVIDER_UNSPECIFIED", "LEGACY_DATAPATH", "ADVANCED_DATAPATH" - Default value. "LEGACY_DATAPATH"
    # https://cloud.google.com/kubernetes-engine/docs/how-to/dataplane-v2
    datapathProvider: ADVANCED_DATAPATH
    networkPolicy:
      # TODO: Does this only enable Calico network policies? If so that is useless.
      enabled: true
  writeConnectionSecretToRef:
    name: auto-pilot-gke-cluster-0
    namespace: crossplane-system
```

4. Once creation is complete, extract `kubeconfig` from secret.

<!-- TODO: Only extract the data into a kubeconfig manifest for simplicity -->
```bash
kubectl get secret auto-pilot-gke-cluster-0 -n crossplane-system -oyaml > kubeconfig-0
```

5. Bootstrap FluxCD into newly created Management GKE Cluster.

```bash
task flux CLUSTER=0
```

### 2. (OPTIONAL) Create the Management GKE cluster via alternative method(s); EG. Terraform
