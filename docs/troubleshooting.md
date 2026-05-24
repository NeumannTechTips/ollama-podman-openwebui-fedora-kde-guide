# Troubleshooting Guide

This document covers the most common issues encountered when following the
installation guide for Ollama + Podman + Open WebUI on Fedora 44 KDE Plasma
with an NVIDIA RTX GPU.

Work through the relevant section methodically. The majority of failures
fall into one of four categories: driver loading, SELinux policy, CDI
specification staleness, or container networking.

---

## Table of Contents

- [NVIDIA Driver Issues](#nvidia-driver-issues)
- [CDI and GPU Passthrough Issues](#cdi-and-gpu-passthrough-issues)
- [SELinux Issues](#selinux-issues)
- [Podman and Container Issues](#podman-and-container-issues)
- [Ollama Issues](#ollama-issues)
- [Open WebUI Issues](#open-webui-issues)
- [Networking Issues](#networking-issues)
- [After a System Update](#after-a-system-update)

---

## NVIDIA Driver Issues

### `nvidia-smi` returns `command not found` after reboot

The driver was not installed, or the kernel module has not loaded.

```bash
# Confirm the driver package is installed
rpm -qa | grep akmod-nvidia

# Check whether the kernel module is loaded
lsmod | grep nvidia

# If not loaded, attempt to load it manually
sudo modprobe nvidia

# If modprobe fails, force akmods to rebuild the module
sudo akmods --force

# Reboot after a forced rebuild
sudo reboot
```

---

### `modinfo -F version nvidia` returns `ERROR: Module nvidia not found`

The akmod build has not yet completed. This is normal immediately after
installation and takes up to 5 minutes on some systems.

```bash
# Wait 2 to 5 minutes, then re-run
modinfo -F version nvidia
```

Do not reboot until this command returns a version number such as `595.xx.xx`.

---

### `nvidia-smi` shows the correct GPU but CUDA version shows `N/A`

The `xorg-x11-drv-nvidia-cuda` package may not have been installed.

```bash
sudo dnf install -y xorg-x11-drv-nvidia-cuda
sudo reboot
```

---

### Driver fails to load after a kernel update

The akmod system rebuilds the module automatically on the next boot following
a kernel update. If it fails:

```bash
# Force a rebuild targeting the currently running kernel
sudo akmods --force --kernels $(uname -r)

# Verify the module is now built
modinfo -F version nvidia

# Reboot
sudo reboot
```

---

## CDI and GPU Passthrough Issues

### `podman run` returns `CDI device nvidia.com/gpu=all not found`

The CDI specification either does not exist or is stale.

```bash
# Check whether the specification file exists
ls -lh /etc/cdi/nvidia.yaml

# Regenerate it
sudo mkdir -p /etc/cdi
sudo nvidia-ctk cdi generate --output=/etc/cdi/nvidia.yaml

# Verify the GPU is listed
nvidia-ctk cdi list
```

Expected output from `nvidia-ctk cdi list`:

```
nvidia.com/gpu=0
nvidia.com/gpu=all
```

---

### `Failed to initialize NVML: Insufficient permissions`

This is almost always an SELinux enforcement issue. See the
[SELinux Issues](#selinux-issues) section below.

---

### GPU is visible on the host but not inside the container

```bash
# Step 1: Confirm CDI spec is current
nvidia-ctk cdi list

# Step 2: Confirm SELinux boolean is enabled
getsebool container_use_devices

# Step 3: Confirm the NVIDIA kernel module is loaded
lsmod | grep nvidia

# Step 4: Run a manual test with explicit SELinux type
podman run --rm \
  --device nvidia.com/gpu=all \
  --security-opt label=type:nvidia_container_t \
  docker.io/nvidia/cuda:12.3.0-base-ubuntu22.04 \
  nvidia-smi
```

---

## SELinux Issues

### Container receives AVC denial when accessing GPU devices

Fedora 44 runs SELinux in Enforcing mode by default. The
`container_use_devices` boolean must be enabled persistently.

```bash
# Check current SELinux mode
getenforce

# Enable the boolean persistently (-P flag is mandatory)
sudo setsebool -P container_use_devices on

# Confirm it is set
getsebool container_use_devices
```

Expected output:

```
container_use_devices --> on
```

---

### Viewing recent SELinux denials related to NVIDIA

```bash
sudo ausearch -m avc -ts recent | grep nvidia
```

If denials are present and the `container_use_devices` boolean is already
enabled, install the NVIDIA Container Toolkit SELinux policy package:

```bash
sudo dnf install -y nvidia-container-toolkit-selinux
```

---

## Podman and Container Issues

### `podman-compose up` fails with `Error: no such image`

The image has not yet been pulled. Allow Podman to pull it on first run,
or pull manually:

```bash
podman pull docker.io/ollama/ollama:latest
podman pull ghcr.io/open-webui/open-webui:main
```

---

### A container exits immediately after starting

```bash
# Check the container logs for the specific error
podman logs ollama
podman logs open-webui
```

Common causes and fixes:

| Error in logs | Likely cause | Fix |
|---------------|-------------|-----|
| `CDI device not found` | Stale CDI spec | Regenerate CDI spec |
| `permission denied /dev/nvidia*` | SELinux blocking | Enable `container_use_devices` boolean |
| `address already in use` | Port conflict | Check `ss -tlnp \| grep 11434` and stop the conflicting process |

---

### `podman ps` shows containers as `Exited`

```bash
# View exit code and last error
podman inspect ollama | grep -A5 '"Status"'

# Follow logs to see what happened at exit
podman logs --tail 50 ollama
```

---

### Volumes are not persisting between restarts

Confirm the volumes exist:

```bash
podman volume ls
```

Expected output:

```
DRIVER      VOLUME NAME
local       ollama-data
local       open-webui-data
```

If either volume is missing, it was likely removed with `podman-compose down -v`.
The `-v` flag deletes volumes. Use `podman-compose down` (without `-v`) to
stop containers without destroying data.

---

## Ollama Issues

### Models are not using the GPU (running on CPU only)

Check the Ollama logs while a model is loading:

```bash
podman logs -f ollama
```

Look for a line similar to:

```
llama_model_load: offloaded 33/33 layers to GPU
```

If instead you see `offloaded 0/33 layers to GPU`, the GPU is not accessible
to the container. Revisit the CDI and SELinux sections above.

---

### `ollama pull` is slow or stalls

Large models (7B and above) are several gigabytes in size. A stall during
download is usually a network issue rather than an Ollama problem.

```bash
# Check download progress inside the container
podman exec -it ollama ollama pull llama3.1:8b

# If a download is interrupted, simply re-run the same command
# Ollama resumes incomplete downloads automatically
```

---

### `ollama list` shows no models

Models are stored in the `ollama-data` volume. If the volume was recreated,
models must be pulled again.

```bash
podman exec -it ollama ollama list
```

---

### Out of VRAM error when loading a model

The model is too large for your GPU VRAM. Options:

1. Use a smaller or more aggressively quantised variant (e.g., `:8b-q4_K_M`)
2. Use `OLLAMA_MAX_LOADED_MODELS=1` in the compose environment to ensure only one model is loaded at a time

```bash
podman exec -it ollama ollama rm <large-model>
podman exec -it ollama ollama pull llama3.1:8b-instruct-q4_K_M
```

---

## Open WebUI Issues

### Open WebUI loads but shows `Could not connect to Ollama`

```bash
# Step 1: Confirm Ollama is healthy
podman inspect ollama | grep -A3 '"Health"'

# Step 2: Test the Ollama API from the host
curl http://localhost:11434/api/tags

# Step 3: Test the API from inside the Open WebUI container
podman exec -it open-webui curl http://ollama:11434/api/tags
```

If Step 2 succeeds but Step 3 fails, the container network is not resolving
the `ollama` hostname. Confirm both containers are part of the same Compose
network by checking `podman network ls` and `podman inspect open-webui`.

---

### Open WebUI is inaccessible at `http://localhost:3000`

```bash
# Confirm the container is running and the port is mapped
podman ps | grep open-webui

# Confirm the port is listening on the host
ss -tlnp | grep 3000
```

---

### First-time sign-up page does not appear

Ensure `ENABLE_SIGNUP=true` is set in the `compose.yml` environment block for
the `open-webui` service. After creating the first administrator account,
set this to `false` and recreate the container to prevent further
self-registration.

---

## Networking Issues

### Containers cannot resolve each other by name

Podman Compose creates a shared network automatically. If name resolution
fails, the network may not have been created correctly.

```bash
# List all Podman networks
podman network ls

# Inspect the compose network
podman network inspect ollama-podman_default

# Confirm both containers are attached
podman inspect ollama | grep -A10 '"Networks"'
podman inspect open-webui | grep -A10 '"Networks"'
```

If either container is missing from the network, bring the stack down and
back up cleanly:

```bash
cd ~/ollama-podman
podman-compose down
podman-compose up -d
```

---

## After a System Update

After running `sudo dnf update`, the following steps should be taken if the
kernel or NVIDIA driver was updated:

```bash
# Step 1: Confirm the driver version after reboot
nvidia-smi

# Step 2: Regenerate the CDI specification
sudo nvidia-ctk cdi generate --output=/etc/cdi/nvidia.yaml

# Step 3: Verify GPU passthrough still works
podman run --rm \
  --device nvidia.com/gpu=all \
  --security-opt label=type:nvidia_container_t \
  docker.io/nvidia/cuda:12.3.0-base-ubuntu22.04 \
  nvidia-smi

# Step 4: Restart the Ollama stack
cd ~/ollama-podman
podman-compose down
podman-compose up -d
```

Skipping Step 2 after a driver update is the single most common cause of
GPU passthrough failures following a system update.

---

*If your issue is not covered here, please open a
[GitHub issue](../../issues/new/choose) using the Bug Report template.*
