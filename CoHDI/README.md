# CoHDI Helm Chart

# Introduction

The **CoHDI Helm chart** deploys the CoHDI system - an integration
layer for CDI management and device configuration in Kubernetes.

## Prerequisite environmental information

The following environment is required.

* K8s v1.34 or higher
* Enable feature gate for DRADeviceTaints
* Enable feature gate for DRADeviceBindingConditions
* Enable feature gate for DRAResourceClaimDeviceStatus

# Getting Started

## Installing the GPU driver

The GPU driver installation process depends on your environment.

- [SLES Instructions](./docs/GPU_DRIVER_SLES.md)
- [RHEL Instructions](./docs/GPU_DRIVER_RHEL.md) (in preparation)

## Setting the provider ID

For FSAS's PRIMERGY CDI, you need to set the provider ID of the Node
resource in the following format:

```
providerID: fsas-cdi://<machine_uuid>

For example:

providerID: fsas-cdi://be85a638-9c81-4ca7-84f7-59ae78bfd672
```

Once you set the provider ID, it cannot be changed later, so please
set it carefully.

## Obtain the Chart

Add the CoHDI Helm repository:

```bash
helm repo add cohdi_helm <link>
helm repo update
```

Or clone directly:

```bash
git clone <link>
cd cohdi_helm
```

## Creating Certificates (for webhook)

These steps show how to create a server private key, CA certificate,
and server certificate for webhooks with OpenSSL.

1. Create a server private key

```bash
openssl genrsa -out tls.key 2048
```

Replace the `common.webhook.server.key` value in 
`CoHDI/charts/composable-resource-operator/values.yaml` of the CoHDI
Helm Chart with the contents of the generated `tls.key` file.

2. Create a CA certificate and a server certificate

```bash
cro=composable-resource-operator
crows=$cro-webhook-service
cross=$crows.$cro-system.svc
openssl req -x509 -key tls.key -new -sha256 -days 365 -out tls.crt \
    -subj "/CN=$crows" \
    -addext "subjectAltName = DNS:$cross, DNS:$cross.cluster.local" \
    -addext "basicConstraints = critical, CA:FALSE" \
    -addext "keyUsage = critical, digitalSignature, keyEncipherment" \
    -addext "extendedKeyUsage = serverAuth, clientAuth"
```

Replace the `common.webhook.client.caBundle` and
`common.webhook.server.crt` values in
`CoHDI/charts/composable-resource-operator/values.yaml` of the CoHDI
Helm Chart with the contents of the generated `tls.crt` file.

## Configure `values.yaml`

There are some pieces of information that need to be entered by the
user.  Update `values.yaml` with the settings.

The `values.yaml` that needs to be updated is below.

```
CoHDI/
  values.yaml (common to all components)
  charts/
    composable-dra-driver/
      values.yaml (for composable-dra-driver)
    composable-resource-operator/
      values.yaml (for composable-resource-operator)
    dynamic-device-scaler/
      values.yaml (for dynamic-device-scaler)
```

Below is a list of items that should be updated in `values.yaml`.

`fti_cdi` corresponds to FSAS's PRIMERGY CDI.

### Common `values.yaml` for all components

| Key                             | Description                                | Get  | Initial value                            |
|:--------------------------------|:-------------------------------------------|:----:|:-----------------------------------------|
| (Vendor-independent parameters) |                                            |      |                                          |
| `deviceInfo`                    | Definition information for each device     | *7   |                                          |
| `fabricIdRange`                 | CDI Fabric ID Range                        | *7   | `"[0]"`                                  |
| (Vendor-dependent parameters [fti_cdi]) |                                    |      |                                          |
| `username`                      | Tenant administrator username              | *2   | `"username"`                             |
| `password`                      | Tenant administrator password              | *2   | `"password"`                             |
| `realm`                         | relm for token acquisition                 | *2   | `"00000000-0000-0000-0000-000000000000"` |
| `client_id`                     | client_id for token acquisition            | *1   | `"cdi"`                                  |
| `client_secret`                 | client_secret for token acquisition        | *1   | `"XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX"`     |
| `CDI_ENDPOINT`                  | FSAS's PRIMERGY CDI API server URL         | *1   | `"https://cdimgr.localdomain"`           |
| `CLUSTER_ID`                    | Cluster UUID                               | *6   | `""`                                     |
| `TENANT_ID`                     | Tenant UUID                                | *2   | `"00000000-0000-0000-0000-000000000000"` |
| `certificate`                   | PEM-encoded CA certificate for validating FSAS PRIMERGY CDI's API server | *1   |            |
| `ip`                            | FSAS's PRIMERGY CDI management IP address  | *1   | `198.51.100.2`                           |
| `hostnames`                     | FSAS's PRIMERGY CDI admin hostname         | *1   | `cdimgr.localdomain`                     |

### `values.yaml` for composable-dra-driver

| Key                             | Description                                | Get  | Initial value                            |
|:--------------------------------|:-------------------------------------------|:----:|:-----------------------------------------|
| (Vendor-independent parameters) |                                            |      |                                          |
| `name`                          | Container name                             | *5   | `composable-dra-driver`                  |
| `image`                         | Container image                            | *4   |                                          |
| `SCAN_INTERVAL`                 | Loop processing interval (sec)             | *4   | `"5s"`                                   |
| (Vendor-dependent parameters [fti_cdi]) |                                    |      |                                          |
| `USE_CAPI_BMH`                  | Availability of ClusterAPI and BMH         | *5   | `"false"`                                |
| `USE_CM`                        | Availability of CM (Cluster Manager)       | *5   | `"false"`                                |

### `values.yaml` for composable-resource-operator

| Key                             | Description                                | Get  | Initial value                            |
|:--------------------------------|:-------------------------------------------|:----:|:-----------------------------------------|
| (Vendor-independent parameters) |                                            |      |                                          |
| `name`                          | Container name                             | *5   | `composable-resource-operator`           |
| `image`                         | Container image                            | *4   |                                          |
| `caBundle`                      | Webhook server CA certificate bundle       | *3   |                                          |
| `crt`                           | TLS crt                                    | *3   |                                          |
| `key`                           | TLS key                                    | *3   |                                          |
| `DEVICE_RESOURCE_TYPE`          | Device resource type                       | *5   | `"DRA"`                                  |
| `CDI_PROVIDER_TYPE`             | CDI provider type                          | *5   | `"FTI_CDI"`                              |
| (Vendor-dependent parameters [fti_cdi]) |                                    |      |                                          |
| `FTI_CDI_API_TYPE`              | FSAS's PRIMERGY CDI API type               | *5   | `"FM"`                                   |

### `values.yaml` for dynamic-device-scaler

| Key                             | Description                                | Get  | Initial value                            |
|:--------------------------------|:-------------------------------------------|:----:|:-----------------------------------------|
| (Vendor-independent parameters) |                                            |      |                                          |
| `name`                          | Container name                             | *5   | `dynamic-device-scaler`                  |
| `image`                         | Container image                            | *4   |                                          |
| `SCAN_INTERVAL`                 | Interval (sec) of periodic timer events    | *4   | `"60"`                                   |
| `DEVICE_NO_REMOVAL_DURATION`    | Time from last use of device until it can be detached | *4 | `"600"`                         |
| `DEVICE_NO_ALLOCATION_DURATION` | Time from last use of device to reschedule | *4   | `"60"`                                   |

```
*1: Check with the FSAS's PRIMERGY CDI system administrator
*2: Tenant-specific information issued by the system administrator to the tenant administrator
*3: User-created
*4: User can set it arbitrarily
*5: Cannot be changed
*6: Not required in SLES environment
*7: See the section on "Setting definition information for each device"
```

### Setting definition information for each device

In this README, the software that enables the addition and deletion of
hardware is referred to as "CoHDI management software".   FSAS calls
it "CDI management software."

To configure this,collect the following information. Currently, only
NVIDIA GPUs are supported.

#### fabricIdRange

* Obtain a list of fabric numbers from the CoHDI management software.
  For FSAS's PRIMERGY CDI, you can obtain this using the
  `cdictl unify fabric list` command.
* Specify the list in array format.  For example, if the fabric
  numbers are 0 and 1, the following is a configuration example.

```
global:
  common:
    configData:
      deviceInfo: omitted
      fabricIdRange: "[0,1]"
```

#### deviceInfo

Collect the following information for all models of installed NVIDIA
GPUs that will be used with the CoHDI management software.

(1) The model name of the GPU registered in the CoHDI management
    software.  For FSAS's PRIMERGY CDI, you can obtain it with the
    `cdictl spec list` command (e.g., "L40S").  
(2) The product name of the NVIDIA GPU (e.g., "NVIDIA L40S").  
    Reference:https://github.com/NVIDIA/open-gpu-kernel-modules  
(3) A unique and arbitrary identifier for K8s (e.g., "nvidia-l40s").
    Choose an appropriate name.  It must follow the rules for DNS
    label names.  
    Reference:https://kubernetes.io/docs/concepts/overview/working-with-objects/names/#dns-label-names

deviceInfo is defined as an array in the following format.

```
deviceInfo: |
  - index: <index number>
    cdi-model-name: "<Info. on (1)>"
    dra-attributes:
      productName: "<Info. on (2)>"
      type: "gpu"
      uuid: ""
    driver-name: "gpu.nvidia.com"
    k8s-device-name: "<Info. on (3)>"
    cannot-coexist-with: "An array listing all index numbers except me"
  - index: <index number>
        :
        :
```

Each entry in the deviceInfo array has the following keys: index,
cdi-model-name, dra-attributes, driver-name, k8s-device-name, and
cannot-coexist-with.  For each resource type (model), an index number
is assigned sequentially, starting from 1, and the deviceInfo
information is constructed using the (1), (2), and (3) above.  When
using an NVIDIA GPU, the driver-name is `gpu.nvidia.com`.

An example of the settings is as follows.

```
global:
  common:
    configData:
      deviceInfo: |
        - index: 1
          cdi-model-name: "a100"
          dra-attributes:
            productName: "NVIDIA A100 80GB PCIe"
            type: "gpu"
            uuid: ""
          driver-name: "gpu.nvidia.com"
          k8s-device-name: "nvidia-a100-80"
          cannot-coexist-with: [2,3]
        - index: 2
          cdi-model-name: "L40S"
          dra-attributes:
            productName: "NVIDIA L40S"
            type: "gpu"
            uuid: ""
          driver-name: "gpu.nvidia.com"
          k8s-device-name: "nvidia-l40s"
          cannot-coexist-with: [1,3]
        - index: 3
          cdi-model-name: "a30"
          dra-attributes:
            productName: "NVIDIA A30"
            type: "gpu"
            uuid: ""
          driver-name: "gpu.nvidia.com"
          k8s-device-name: "nvidia-a30"
          cannot-coexist-with: [1,2]
```

# Installation and simple operation check

## Install/Upgrade CoHDI

```bash
helm upgrade --install cohdi . -n cohdi --create-namespace
```

## Check status

```bash
kubectl get pods -A
```

## Verifying installation completion

Verify that the following three Pods exist and are Running.

```
$ kubectl get pods -A
NAMESPACE                             NAME                                                              READY   STATUS    RESTARTS   AGE
composable-dra                        composable-dra-driver-b87cc77d7-7mwg9                             1/1     Running   0          112m
composable-dra                        dds-controller-manager-c777b4f95-dxc2q                            1/1     Running   0          18m
composable-resource-operator-system   composable-resource-operator-controller-manager-79c58887bcjbl99   1/1     Running   0          6h20m
...
```

To check the operation of each CoHDI Pod after installation, do the
following.

### composable-dra-driver

```
$ kubectl logs -n composable-dra composable-dra-driver-xxx
    :
    :
time=2025-12-10T23:25:43.011Z level=INFO source=manager.go:174 msg="Loop Successful" compo=CDI_DRA
    :
    :
```

* Make sure that the message `Loop Successful` is output to the log
  and that there are no error messages.

```
$ kubectl get resourceslice
NAME                   NODE   DRIVER           POOL                     AGE
gpu.nvidia.com-96qth          gpu.nvidia.com   nvidia-l40s-fabric0      2m3s
gpu.nvidia.com-mm9jg          gpu.nvidia.com   nvidia-a100-80-fabric0   2m3s
gpu.nvidia.com-mxx46          gpu.nvidia.com   nvidia-a30-fabric0       2m3s
```

* Make sure you create a ResourceSlice without any node information.

### composable-resource-operator

```
$ kubectl logs -n composable-resource-operator-system composable-resource-operator-controller-manager-xxx
2025-12-10T23:28:20Z    INFO    upstream_syncer_controller      start scheduled upstream data synchronization
2025-12-10T23:28:20Z    INFO    fti_fm_client   start getting resources
2025-12-10T23:28:21Z    INFO    upstream_syncer_controller      Device is still missing but within grace period, skipping       {"deviceID": "GPU-a83da4d9-d250-ff86-30e7-352b57eb9202"}
```

* Make sure there are no errors in the log.

### dynamic-device-scaler

```
$ kubectl logs -n composable-dra dds-controller-manager-xxx
{"level":"info","ts":"2025-12-10T23:30:42Z","logger":"DDS","msg":"Start reconcile"}
```

* Make sure that the message `Start reconcile` is output to the log
  and that there are no error messages.

## Check available resources

In the context of K8s DRA, resources available within a K8s cluster
are output as a resource called a ResourceSlice.  The manager source
(output source) of the ResourceSlice changes depending on whether it
is attached to a node.

### Resources attached to the node

For resources already attached to a node, the driver provided by the
device vendor outputs a ResourceSlice.  A characteristic of this is
that a NodeName exists under Spec and has a value (in the example,
worker-2odah).  For example, for an NVIDIA GPU, the DRA driver
provided by NVIDIA outputs the following information:

```
$ kubectl describe resourceslices
Name:         worker-2odah-gpu.nvidia.com-5lztt
Namespace:
Labels:       <none>
Annotations:  <none>
API Version:  resource.k8s.io/v1
Kind:         ResourceSlice
Metadata:
  Creation Timestamp:  2025-11-14T13:29:08Z
  Generate Name:       worker-2odah-gpu.nvidia.com-
  Generation:          1
  Owner References:
    API Version:     v1
    Controller:      true
    Kind:            Node
    Name:            worker-2odah
    UID:             ba79b856-fa88-41f3-80ac-52f0ff24d184
  Resource Version:  13856105
  UID:               63801c3a-cb61-446e-acd9-9589ab65c8c0
Spec:
  Devices:
    Attributes:
      Architecture:
        String:  Ada Lovelace
      Brand:
        String:  Nvidia
      Cuda Compute Capability:
        Version:  8.9.0
      Cuda Driver Version:
        Version:  13.0.0
      Driver Version:
        Version:  580.82.7
      Index:
        Int:  0
      Minor:
        Int:  0
      Pcie Bus ID:
        String:  0000:97:00.0
      Product Name:
        String:  NVIDIA L40S
      resource.kubernetes.io/pcieRoot:
        String:  pci0000:8e
      Type:
        String:  gpu
      Uuid:
        String:  GPU-3b8868ca-2af5-fffb-ab1f-ccafa2ce04cc
    Capacity:
      Memory:
        Value:  46068Mi
    Name:       gpu-0
  Driver:       gpu.nvidia.com
  Node Name:    worker-2odah
  Pool:
    Generation:            1
    Name:                  worker-2odah
    Resource Slice Count:  1
Events:                    <none>
```

### Resources not attached to a node

For resources that are not attached to a node, the cdi-dra pod
included in CoHDI outputs a ResourceSlice. Specifically, it outputs
the following information.  A notable feature is that a "Node
Selector" exists under Spec and is filled with a value.  The number of
Name: entries under Spec.Devices is the number of devices that are
available to K8s but are not attached to a node (in this example,
there are two L40S resources that are not attached to the node).

```
$ kubectl describe resourceslices
Name:         gpu.nvidia.com-dvttx
Namespace:
Labels:       <none>
Annotations:  <none>
API Version:  resource.k8s.io/v1
Kind:         ResourceSlice
Metadata:
  Creation Timestamp:  2025-11-14T11:39:54Z
  Generate Name:       gpu.nvidia.com-
  Generation:          5
  Resource Version:    13863574
  UID:                 ef538b2a-bb7a-4a8f-99c6-9701bef5af97
Spec:
  Devices:
    Attributes:
      Product Name:
        String:  NVIDIA L40S
      Type:
        String:  gpu
    Binding Conditions:
      FabricDeviceReady
    Binding Failure Conditions:
      FabricDeviceReschedule
      FabricDeviceFailed
    Binds To Node:  true
    Name:           nvidia-l40s-gpu0
    Attributes:
      Product Name:
        String:  NVIDIA L40S
      Type:
        String:  gpu
    Binding Conditions:
      FabricDeviceReady
    Binding Failure Conditions:
      FabricDeviceReschedule
      FabricDeviceFailed
    Binds To Node:  true
    Name:           nvidia-l40s-gpu1
  Driver:           gpu.nvidia.com
  Node Selector:
    Node Selector Terms:
      Match Expressions:
        Key:       cohdi.io/nvidia-l40s
        Operator:  In
        Values:
          true
        Key:       cohdi.io/fabric
        Operator:  In
        Values:
          0
  Pool:
    Generation:            5
    Name:                  nvidia-l40s-fabric0
    Resource Slice Count:  1
Events:                    <none>
```

## How to deploy workload

This procedure assumes that you have an actual device such as a
FSAS's PRIMERGY CDI.

In a Kubernetes environment, workloads are launched in the form of
Pods.  Pods can define resource requirements according to the K8s
Dynamic Resource Allocation mechanism.  To achieve dynamic scaling
using CoHDI, Pods are launched in accordance with this DRA mechanism.

https://kubernetes.io/docs/concepts/scheduling-eviction/dynamic-resource-allocation/

The Pods are deployed in two steps.

1. Create a ResourceClaimTemplate
2. Launch the Pod by referencing the created ResourceClaimTemplate

### Create a ResourceClaimTemplate

Currently, we are assuming usage that uses the productName output by
the NVIDIA driver.
For example, if you want to use an NVIDIA GPU with a productName of
"NVIDIA L40S", create a ResourceClaimTemplate with the following
definition (assuming that is defined in the deviceInfo above).

[ResourceClaimTemplate-workload.yaml]
```
apiVersion: resource.k8s.io/v1
kind: ResourceClaimTemplate
metadata:
  name: single-gpu
spec:
  spec:
    devices:
      requests:
      - name: gpu
        firstAvailable:
          - name: node-gpu
            allocationMode: ExactCount
            count: 1
            deviceClassName: gpu.nvidia.com
            selectors:
            - cel:
                expression: |-
                   device.attributes["gpu.nvidia.com"].productName == "NVIDIA L40S" &&
                   device.attributes["gpu.nvidia.com"].uuid != ""
          - name: fabric-gpu
            allocationMode: ExactCount
            count: 1
            deviceClassName: gpu.nvidia.com
            selectors:
            - cel:
                expression: |-
                   device.attributes["gpu.nvidia.com"].productName == "NVIDIA L40S"
```

```bash
kubectl apply -f ResourceClaimTemplate-workload.yaml
```

### Launch the Pod by referencing the created ResourceClaimTemplate

Launch a Pod so that it is linked to the ResourceClaimTemplate you
created.  Here, we use the ResourceClaimTemplate name `single-gpu` to
link the ResourceClaimTemplate to the Pod.

[Pod-workload.yaml]
```
apiVersion: v1
kind: Pod
metadata:
  name: workload-pod-1
spec:
  containers:
  - name: workload
    image: nginx
    resources:
      claims:
      - name: gpu
  resourceClaims:
  - name: gpu
    resourceClaimTemplateName: single-gpu
```

```bash
kubectl apply -f Pod-workload.yaml
```

This will create a Pod with the claims defined in the
ResourceClaimTemplate.

## Checking the allocation status to Pods

The Pod will be in the Pending state as shown below until the
resources attached to the node are actually assigned to the Pod and
the Pod can run on the node, such as waiting for the resources to be
attached.

```
$ kubectl get pods
NAME                READY   STATUS    RESTARTS   AGE
workload-pod-1      0/1     Pending   0          10s
```

When the Pod is ready to start, it is expected to go through the
following ContainerCreating states before entering the Running state.

```
$ kubectl get pods
NAME                READY   STATUS              RESTARTS   AGE
workload-pod-1      0/1     ContainerCreating   0          22s
```

A Running state means that the Pod is running using the resources it
requested.

```
$ kubectl get pods
NAME                READY   STATUS    RESTARTS   AGE
workload-pod-1      1/1     Running   0          83s
```

To check whether a GPU has actually been allocated to a Pod, check the
information under Allocation as the Status of the ResourceClaim, as
shown below.

```
$ kubectl describe resourceclaim
Name:         xxx
API Version:  resource.k8s.io/v1
Kind:         ResourceClaim
...
Status:
  Allocation:  # A list of resources actually assigned to resource requests
    Allocation Timestamp:  2025-11-12T12:48:35Z
    Devices:
      Results:
        Device:   gpu-0  # Name of the assigned GPU (Name written in ResourceSlice)
        Driver:   gpu.nvidia.com
        Pool:     worker-2odah
        Request:  gpu
    Node Selector:
      Node Selector Terms:
        Match Fields:
          Key:       metadata.name
          Operator:  In
          Values:
            worker-2odah  # Node Name
  Reserved For:
    Name:      workload-pod-1  # The name of the Pod using this ResourceClaim
    Resource:  pods
    UID:       aa7e6a81-7914-4413-ab40-4bfb4c0d846a
```

## Uninstall

If uninstalling CoHDI while there is a ComposableResource in progress,
problems may occur.  To be safe, run the following command to confirm
that `No resources found` is output and there is no
ComposableResource.  Then run the uninstall command.

```
$ kubectl get composableresource -A
No resources found
```

```bash
helm uninstall cohdi -n cohdi
```
