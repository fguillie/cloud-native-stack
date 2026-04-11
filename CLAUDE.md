# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

NVIDIA Cloud Native Stack (CNS) is an Ansible-based deployment automation framework for Kubernetes clusters with NVIDIA GPU support. It manages the full lifecycle of GPU-accelerated Kubernetes clusters across bare metal, cloud providers, and edge devices.

## Common Commands

All commands are run from the `playbooks/` directory.

```bash
# Install CNS (bare metal / local)
bash setup.sh install

# Cloud provider installs (uses Terraform)
bash setup.sh install gke
bash setup.sh install eks
bash setup.sh install aks

# Confidential Computing install
bash setup.sh install cc

# Lifecycle management
bash setup.sh validate    # Post-install validation only
bash setup.sh upgrade     # Auto-detects current version and upgrades
bash setup.sh uninstall
bash setup.sh uninstall [gke|eks|aks]
```

Direct Ansible invocations (skip setup.sh):
```bash
ansible-playbook -i hosts cns-validation.yaml
ansible-playbook -i hosts cns-upgrade.yaml
ansible-playbook -i hosts cns-uninstall.yaml
ansible-playbook -c local -i localhost, cns_cc_bios.yaml   # Confidential Computing BIOS
```

## Architecture

### Entry Point and Configuration Flow

1. **`setup.sh`** — main entry point; installs Ansible dependencies, parses arguments, triggers playbooks
2. **`cns_version.yaml`** — sets the active CNS version (e.g., `cns_version: 16.1`)
3. **`cns_values_<version>.yaml`** — version-specific component versions and feature flags (sourced automatically from `cns_version.yaml`)
4. **`hosts`** — Ansible inventory; must be updated with target node IPs before running

### Playbook Orchestration

`cns-installation.yaml` is the main orchestrator. It imports sub-playbooks in order:

```
cns-installation.yaml
  ├── prerequisites.yaml        # K8s repos, kubelet/kubeadm/kubectl install
  ├── nvidia-driver.yaml        # NVIDIA driver installation (if cns_nvidia_driver == true)
  ├── nvidia-docker.yaml        # Docker daemon config (developer stack only)
  ├── k8s-install.yaml          # kubeadm cluster init, Calico/Flannel CNI
  ├── operators-install.yaml    # GPU Operator, Network Operator, NIM, Nsight, KAI Scheduler
  ├── add-ons.yaml              # Storage, monitoring, KServe, LWS, Volcano, MetalLB
  └── cns-validation.yaml       # Post-install component version checks
```

Other top-level playbooks:
- `cns-upgrade.yaml` — detects running Kubernetes version, maps to CNS version, upgrades in-place
- `cns-uninstall.yaml` — full teardown (operators → cluster → drivers)
- `microk8s.yaml` — lightweight alternative to full kubeadm cluster (used for Jetson/edge)

### Version and Component Management

- **`cns.json`** — master manifest of all CNS versions and their exact component versions; source of truth for the entire version matrix
- **`gpu_operator.yaml`** — maps GPU Operator releases to sub-component versions (driver_manager, container_toolkit, device_plugin, dcgm_exporter, nfd, gfd, mig_manager, validator)
- **`cns_values_*.yaml`** — one file per supported version; contains all Helm chart versions, image tags, and feature toggle defaults

### Feature Flags (in `cns_values_<version>.yaml`)

All default to `no`/`false` unless noted:

| Flag | Purpose |
|---|---|
| `install_k8s: yes` | Install Kubernetes (enabled by default) |
| `enable_gpu_operator: yes` | Deploy GPU Operator (enabled by default) |
| `container_runtime` | `containerd` (default) or `cri-o` |
| `confidential_computing` | AMD SNP Confidential Computing |
| `enable_mig` | GPU Multi-Instance GPU mode |
| `enable_gds` | GPUDirect Storage |
| `enable_vgpu` | vGPU support (requires gridd.conf license) |
| `enable_network_operator` | NVIDIA Network Operator + Mellanox |
| `enable_rdma` | RDMA / RoCE support |
| `enable_kai_scheduler` | KAI GPU scheduler |
| `enable_nim_operator` | NIM Operator for inference |
| `enable_nsight_operator` | NVIDIA Nsight Operator |
| `microk8s` | Use MicroK8s instead of kubeadm |
| `storage` | Local Path / NFS provisioners |
| `monitoring` | Prometheus + Grafana stack |
| `kserve` | KServe with Istio + Knative |
| `loadbalancer` | MetalLB |
| `lws` | LeaderWorkerSet |
| `volcano` | Volcano batch scheduler |

### Supporting Files (`playbooks/files/`)

Configuration templates used by playbooks:
- `network-operator-values.yaml` / `nic-cluster-policy.yaml` — Network Operator Helm values
- `kube-prometheus-stack.values` / `grafana.yaml` — monitoring stack values
- `csp_install.yaml` / `csp_values.yaml` — cloud provider integration
- `nsight_custom_values.yaml` — Nsight Operator custom values
- `gridd.conf` — NVIDIA GRID/vGPU license server config
- `redfish.py` — Redfish BMC API utility
- `aws_credentials` — AWS credentials template (do not commit real credentials)

### Kubernetes Templates (`playbooks/templates/`)

Jinja2 templates for kubeadm cluster initialization:
- `kubeadm-init-config.template` / `kubeadm-init-config-new.template`
- `kubeadm-join.template` — worker node join config
- `metal-lb.template` — MetalLB address pool config

## Supported Platforms

- **OS:** Ubuntu 22.04 LTS, Ubuntu 24.04 LTS, RHEL 8.10
- **Arch:** x86_64, arm64
- **Hardware:** NVIDIA Certified Servers, DGX, Jetson (AGX/NX/Orin), GB200 NVL72
- **Cloud:** GKE, EKS, AKS (via Terraform in `playbooks/guides/Cloud_Guide.md`)

## Hardware-Specific Behavior

- **GB200/GB300:** `cns-installation.yaml` automatically injects GRUB parameters (`iommu.passthrough=1`, `init_on_alloc=0`, `numa_balancing=disable`) when these systems are detected
- **RTX Pro 6000:** HMM is disabled via modprobe parameters
- **Jetson/Tegra:** MicroK8s is the preferred K8s distribution (`microk8s: yes`)
- **Confidential Computing (AMD SNP):** Kernel SNP support is detected automatically; use `bash setup.sh install cc` or run `cns_cc_bios.yaml` for BIOS prep

## Active CNS Versions

CNS maintains 3 versions concurrently. As of April 2026:

| Version | Status | Kubernetes | GPU Operator | Driver |
|---|---|---|---|---|
| 17.1 | Latest GA | 1.34.6 | 26.3.0 | 580.126.20 |
| 17.0 | Maintenance | 1.34.2 | 25.10.1 | 580.105.08 |
| 16.2 | Maintenance | 1.33.10 | 26.3.0 | 580.126.20 |

Older versions are preserved in `playbooks/older_versions/`.

## Upgrade Path

`cns-upgrade.yaml` auto-detects the current cluster's Kubernetes version and maps it to a CNS version:

| K8s Version | CNS Version |
|---|---|
| v1.31.2 | CNS 14.1 |
| v1.31.6 | CNS 14.2 |
| v1.32.2 | CNS 15.1 |
| v1.32.6 | CNS 15.2 |
| v1.33.2 | CNS 16.1 |
| v1.33.6 | CNS 16.2 |
| v1.34.2 | CNS 17.1 |

## Contribution Requirements

All commits require a Developer Certificate of Origin sign-off:
```bash
git commit -s   # adds Signed-off-by: Name <email>
```
