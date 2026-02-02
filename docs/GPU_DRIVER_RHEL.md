The procedure has not been confirmed. It is currently being prepared.

# Installing the GPU Driver on RHEL

## Add NVIDIA CUDA repository

```bash
sudo dnf config-manager --add-repo https://developer.download.nvidia.com/compute/cuda/repos/rhel9/x86_64/cuda-rhel9.repo
sudo dnf clean all
sudo dnf makecache
```

## Install Open Kernel driver KMOD

```bash
sudo dnf install -y nvidia-driver-open
```

## Install utilities

```bash
sudo dnf install -y nvidia-utils nvidia-persistenced
```
