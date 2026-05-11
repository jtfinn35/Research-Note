# NVIDIA Aerial 5G SDK Deployment & Testing Guide

**Status:** Work in Progress
**Author:** Finn
**Environment:** DGX Spark Node (`spark-d957`)

---

## 1. System Architecture & Prerequisites
> 💡 **Note:** Aerial SDK and DPDK integrations are highly sensitive to driver, kernel version matching, and memory page sizes. The following environment matrix documents the exact state of the DGX Spark node and client environment before deployment.

### 1.1 Hardware Specifications
* **Server Node:** NVIDIA DGX Spark Node (`spark-d957`)
* **GPU Processor:** NVIDIA GB10
* **Network Interface Card (NIC):** Mellanox Technologies MT2910 Family [ConnectX-7] (4 Ports)
* **Client Environment:** Acer Predator PHN16-72 (Intel Core i9-14900HX, 32GB RAM) running Windows 11 Home via VS Code Remote-SSH (IP: 140.118.122.123).
* [DGX Spark Node Hardware Environment Verification](https://github.com/jtfinn35/Research-Note/blob/main/photos/screenshots.md#--system-specifications)

### 1.2 Software Environment Matrix
* **Operating System:** Ubuntu 24.04.4 LTS (Noble)
* **Linux Kernel:** `6.14.0-1015-nvidia-64k` *(Note: 64KB page size kernel, critical for DPDK hugepages configuration)*
* **NVIDIA Driver Version:** 590.48.01
* **CUDA Version:** * Runtime API (`nvidia-smi`): 13.1
  * Compiler (`nvcc`): 13.0.88
* **Mellanox OFED Driver:** OFED-internal-25.10-1.7.1

---
