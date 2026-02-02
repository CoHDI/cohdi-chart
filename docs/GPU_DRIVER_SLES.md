# Installing the GPU Driver on SLES

## RKE2 Environment Setup Prerequisites

* The RKE2 environment must be ready and running properly.
* A SUSE subscription must be attached to the target worker node.
* A compatible NVIDIA GPU must be attached to the target worker node.

The following steps are required.

* Add the NVIDIA package repository
* Install the NVIDIA driver and various dependencies
* Install the NVIDIA container toolkit
* Generate the CDI configuration YAML file (`/etc/cdi/nvidia.yaml`)

## Installing the NVIDIA GPU driver on SLES

Refer to the following documentation to install the GPU driver,
container toolkit, etc. on the Worker node.

```
https://documentation.suse.com/suse-ai/1.0/html/NVIDIA-GPU-driver-on-SLES/index.html
```

By performing the steps described in "2.2.1 Installing the NVIDIA GPU
driver on SUSE Linux Enterprise Server", the following packages will
be installed on the worker node.

* nv-prefer-signed-open-driver (NVIDIA Open Driver Package)
* nvidia-compute-utils-G06 (NVIDIA Driver Utilities Package)
* nvidia-persistenced (NVIDIA Driver Persistence Daemon Package)
* nvidia-container-toolkit (NVIDIA Container Toolkit Package)
* cuda-libraries (CUDA Library Package)
* cuda-demo-suite (CUDA Demo Suite Package)

## Additional driver configuration

If the line `options nvidia-drm modeset=1` exists in a file ending in
`.conf` in the `/etc/modprobe.d/` directory, change `modeset=1` to
`modeset=0`.

If this line does not exist, create a new file ending in `.conf` (for
example, `nvidia-drm_modeset.conf`) in the `/etc/modprobe.d/`
directory and write the following line in it.

```
options nvidia-drm modeset=0
```

## Deploying the NVIDIA GPU Operator

Deploy the GPU Operator in the Control Plane by referring to the
following document.

```
https://docs.rke2.io/add-ons/gpu_operators#operator-installation
```

Step 1:

First, install with the default configuration, where each
ClusterPolicy `spec` is set to `enabled: true`.

After installation, it is expected that
`/var/lib/rancher/rke2/agent/etc/containerd/config.toml` will contain
`nvidia` strings.

```bash
sudo grep nvidia /var/lib/rancher/rke2/agent/etc/containerd/config.toml
```

Step 2:

After all the created Pods are Running, edit the ClusterPolicy using
`kubectl edit clusterpolicy` command as follows to turn off components
other than gfd.

```
apiVersion: v1
items:
  spec:
    ccManager:
      enabled: true -> false
    cdi:
      enabled: true -> false
    dcgm:
      enabled: true -> false
    dcgmExporter:
      enabled: true -> false
      serviceMonitor:
        enabled: true -> false
    devicePlugin:
      enabled: true -> false
    driver:
      enabled: true -> false
      rdma:
        enabled: true -> false
    gdrcopy:
      enabled: true -> false
    gds:
      enabled: true -> false
    gfd:
      enabled: true
    kataManager:
      enabled: true -> false
    migManager:
      enabled: true -> false
    nodeStatusExporter:
      enabled: true -> false
    psa:
      enabled: true -> false
    sandboxWorkloads:
      enabled: true -> false
    toolkit:
      enabled: true -> false
    vfioManager:
      enabled: true -> false
    vgpuDeviceManager:
      enabled: true -> false
    vgpuManager:
      enabled: true -> false
```

## Installing DRA

Install DRA in the Control Plane by referring to the following steps.

Get `nvidia-dra-driver-gpu` from `https://helm.ngc.nvidia.com/nvidia`.
Specify `v25.3.2` as the version using `--version` and execute the
following command.  Only this version (`v25.3.2`) has been verified.

```bash
helm repo add nvidia https://helm.ngc.nvidia.com/nvidia/
helm repo update
helm upgrade -i nvidia-dra-driver-gpu nvidia/nvidia-dra-driver-gpu \
    -n nvidia-dra-driver-gpu \
    --create-namespace \
    --version=25.3.2 \
    --set image.tag=v25.3.2 \
    --set nvidiaDriverRoot=/ \
    --set maskNvidiaDriverParams=false \
    --set gpuResourcesEnabledOverride=true \
    --wait
```

After installation, please confirm that the following Dynamic Resource
Allocation DeviceClass exists.

```bash
kubectl get deviceclass gpu.nvidia.com -o yaml
apiVersion: resource.k8s.io/v1
kind: DeviceClass
metadata:
  annotations:
    meta.helm.sh/release-name: nvidia-dra-driver-gpu
    meta.helm.sh/release-namespace: nvidia-dra-driver-gpu
  creationTimestamp: "2026-01-13T12:36:09Z"
  generation: 1
  labels:
    app.kubernetes.io/managed-by: Helm
  name: gpu.nvidia.com
  resourceVersion: "1508263"
  uid: d615b6ea-2b84-4649-aa53-a316f3c24c4a
spec:
  selectors:
  - cel:
      expression: device.driver == 'gpu.nvidia.com' && device.attributes['gpu.nvidia.com'].type
        == 'gpu'
```

Also, please confirm that the following ResourceSlice objects exist.

```
Every 2.0s: kubectl get resourceslices.resource.k8s.io

NAME                                               NODE               DRIVER                      POOL               AGE
controller-qctvl-compute-domain.nvidia.com-dwnbc   controller-qctvl   compute-domain.nvidia.com   controller-qctvl   3m47s
controller-qctvl-gpu.nvidia.com-v8xh2              controller-qctvl   gpu.nvidia.com              controller-qctvl   3m47s
```

## Stop nvidia-persistence

If nvidia-persistenced is running, hot remove may not be possible.
Therefore, stop it and do not start it again.

```bash
sudo systemctl disable nvidia-persistenced.service
sudo systemctl stop nvidia-persistenced.service
```

## Check installation

Verify that the result of the following command is `N`.

```bash
sudo cat /sys/module/nvidia_drm/parameters/modeset
```

Note: This check requires that you have at least one suitable GPU
      connected to the agent node from the LCC.

```bash
sudo reboot
sudo nvidia-smi
```

Note: At this point, if you attach a suitable GPU to the agent node
      from LCC, the GPU will be visible in /sbin/lspci | grep NVIDIA
      and nvidia-smi.

## Limitations

By setting `nvidia-drm` to `modeset=0` in "Additional driver
configuration" and changing `enabled` of each item from `true` to
`false` in "Deploying the NVIDIA GPU Operator", the nvidia-drm modeset
and functions such as dcgm, mig, vgpu will result in restrictions.
