# OAI 5G SA (Standalone) Deployment and Testing Manual

![Version](https://img.shields.io/badge/version-v1.0-blue.svg)
![Status](https://img.shields.io/badge/status-verified-brightgreen.svg)
![OS](https://img.shields.io/badge/OS-Ubuntu_22.04-orange.svg)

> **Author**: 黃仁廷 (JTFinn)
> **Date Created**: 2026-05-03

> ⚠️ **Note**
> All tests in this manual are based on the OAI `develop` branch. Since open-source projects are updated frequently, cloning the repository at a different time might result in discrepancies in core network configuration formats (e.g., YAML) or compilation parameters. Before executing this SOP, please ensure your hardware has at least 24GB of available RAM.

### Table of Contents
1. [Executive Summary](#1-executive-summary)
2. [Architecture and Topology](#2-architecture-and-topology)
3. [Prerequisites](#3-prerequisites)
4. [Step-by-Step Guide](#4-step-by-step-guide)
5. [Configuration](#5-configuration)
6. [Verification](#6-verification)
7. [Troubleshooting](#7-troubleshooting)
8. [References](#8-references)

---

## 1. Executive Summary
This document aims to provide a standardized installation and testing procedure for the OpenAirInterface (OAI) 5G Standalone (SA) mode. It covers the deployment of the OAI Core Network (CN5G) and Radio Access Network (RAN) within a WSL2 environment, utilizing the RF Simulator to achieve an End-to-End (E2E) simulated connection between the Base Station (gNB) and the User Equipment (UE). This establishes a foundational testing platform for the subsequent implementation of PRACH attack and mitigation algorithms.

## 2. Architecture and Topology
This environment utilizes a pure software simulation consisting of the following three main nodes. All connections are established internally via Localhost (127.0.0.1):

*   **OAI-CN5G (5G Core Network)**: Deployed via Docker Compose, comprising essential Network Functions (NFs) such as AMF, SMF, UPF, and NRF.
*   **OAI gNB (Base Station)**: Compiled from the OAI RAN source code. It runs the `nr-softmodem` executable and utilizes the RF Simulator to replace physical USRP hardware interfaces.
*   **OAI nrUE (User Equipment)**: Compiled from the OAI RAN source code. It runs the `nr-uesoftmodem` executable to simulate a 5G SA terminal, establishing the `oaitun_ue1` virtual network interface for data transmission with the core network.

## 3. Prerequisites
To ensure the stability of the OAI system during compilation and execution, and to prevent Out of Memory (OOM) crashes during the parallel compilation of massive C++ projects, the environment of the host machine has been specifically tuned:

*   **Host Machine**: Acer Predator PHN16-72 (32GB RAM, 32 Logical Cores).
*   **Virtualization Environment**: Windows Subsystem for Linux (WSL2) v2.6.3.
    *   **Performance Tuning Strategy**: By default, WSL2 restricts memory allocation to 50% of the physical RAM (16GB). Executing `make -j $(nproc)` (32-thread parallel compilation) under these conditions easily exhausts available memory, resulting in killed processes. Therefore, resources were explicitly reallocated via the Windows `%userprofile%\.wslconfig` file:
        ```ini
        [wsl2]
        memory=24GB
        processors=24
        swap=8GB
        ```
*   **Operating System**: Ubuntu 22.04 LTS (Kernel 6.6.87).
*   **Containerization Configuration (Docker Compose)**:
    *   The default timezone in the official OAI `docker-compose.yaml` is set to Paris (`TZ=Europe/Paris`). To ensure that the log timestamps align with the local environment for subsequent 6G/NTN Scheduling Latency experiments, the environment variables for all core NFs (e.g., MySQL, AMF, SMF) were modified to Taiwan Standard Time:
        ```yaml
        environment:
           - TZ=Asia/Taipei
        ```

---

## 4. Step-by-Step Guide

#### 4.1 Install Docker (5G Core Network Runtime Environment)
```bash
sudo apt update
sudo apt install -y git net-tools putty ca-certificates curl

# Add Docker's official GPG key and repository
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL [https://download.docker.com/linux/ubuntu/gpg](https://download.docker.com/linux/ubuntu/gpg) -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] [https://download.docker.com/linux/ubuntu](https://download.docker.com/linux/ubuntu) $(. /etc/os-release && echo \"${UBUNTU_CODENAME:-$VERSION_CODENAME}\") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

### 4.2 Obtain OAI 5G Core Network (CN5G) Configuration Files
```bash
wget -O ~/oai-cn5g.zip https://gitlab.eurecom.fr/oai/openairinterface5g/-/archive/develop/openairinterface5g-develop.zip?path=doc/tutorial_resources/oai-cn5g
unzip ~/oai-cn5g.zip
mv ~/openairinterface5g-develop-doc-tutorial_resources-oai-cn5g/doc/tutorial_resources/oai-cn5g ~/oai-cn5g
rm -r ~/openairinterface5g-develop-doc-tutorial_resources-oai-cn5g ~/oai-cn5g.zip
```

### 4.3 Install UHD Driver (OAI Compilation Dependency)
Although the RF Simulator is used, the compilation process still depends on the UHD library. An out-of-source build is employed to keep the source directory clean:
```bash
sudo apt install -y autoconf automake build-essential ccache cmake cpufrequtils doxygen ethtool g++ git inetutils-tools libboost-all-dev libncurses-dev libusb-1.0-0 libusb-1.0-0-dev libusb-dev python3-dev python3-mako python3-numpy python3-requests python3-scipy python3-setuptools python3-ruamel.yaml

git clone https://github.com/EttusResearch/uhd.git ~/uhd
cd ~/uhd && git checkout v4.8.0.0
cd host && mkdir build && cd build
cmake ../
make -j $(nproc)
sudo make install
sudo ldconfig
sudo uhd_images_downloader
```

### 4.4 Compile OAI Base Station (gNB) and User Equipment (nrUE)
```bash
# 1. Clone the repository and switch to the develop branch
git clone https://gitlab.eurecom.fr/oai/openairinterface5g.git ~/openairinterface5g
cd ~/openairinterface5g
git checkout develop

# 2. Install dependencies required for the automated build script
cd ~/openairinterface5g/cmake_targets
./build_oai -I
sudo apt install -y libforms-dev libforms-bin

# 3. Execute parallel compilation (including gNB, nrUE, and RF Simulator)
./build_oai -w USRP --ninja --nrUE --gNB --build-lib "nrscope" -C
```

---

## 5. Configuration
This environment is primarily based on the official OAI default configurations. However, to accommodate the routing rules of the WSL2 Docker subnet, a critical modification must be made to the base station (gNB) configuration file to prevent return packets from being dropped by the system as broadcast packets.

*   **gNB Configuration File Path**: `targets/PROJECTS/GENERIC-NR-5GC/CONF/gnb.sa.band78.fr1.106PRB.usrpb210.conf`
*   **Frequency Band and Bandwidth**: Band 78, FR1, 106 PRB.
*   **UE Parameters**: Use the default IMSI `001010000000001` to match the default registered user in the Docker core network database.

**[Modification: Avoiding the Broadcast Address Trap]**
Open the aforementioned gNB configuration file using a text editor (`nano`) and locate the `NETWORK_INTERFACES` section:
Change the default `192.168.70.191` (which is the broadcast address for the `/26` subnet) **to a valid Host IP, such as `192.168.70.129`**.
If this modification is not made, the connection will successfully establish, but the Data Plane (Ping testing) will experience 100% packet loss.

---

## 6. Verification
This section verifies the status of each node within the 5G End-to-End (E2E) connection.

#### 6.1 Core Network (CN5G) Status Check and Firewall Configuration
Before starting, to prevent the local WSL2 firewall from blocking GTP-U forwarding packets, execute the following command to grant forwarding permissions:
```bash
sudo iptables -P FORWARD ACCEPT
```

Next, start the OAI core network containers:
```bash
cd ~/oai-cn5g
docker compose pull
docker compose up -d
docker ps
```
**Expected Output**：
<img width="1807" height="516" alt="螢幕擷取畫面 2026-05-03 003342" src="https://github.com/user-attachments/assets/2bdeb6ad-36e6-4d11-9583-101ab6c4aedb" />

#### 6.2 Starting gNB and nrUE Connection

**Start gNB (Window 1)**:
```bash
cd ~/openairinterface5g/cmake_targets/ran_build/build
sudo ./nr-softmodem -O ../../../targets/PROJECTS/GENERIC-NR-5GC/CONF/gnb.sa.band78.fr1.106PRB.usrpb210.conf --gNBs.[0].min_rxtxtime 6 --rfsim --rfsimulator.[0].serveraddr server
```

Start nrUE (Window 2):
```bash
cd ~/openairinterface5g/cmake_targets/ran_build/build
sudo ./nr-uesoftmodem -O ../../../targets/PROJECTS/GENERIC-NR-5GC/CONF/ue.conf --sa -r 106 --numerology 1 --band 78 -C 3619200000 --ssb 516 --rfsim --rfsimulator.serveraddr 127.0.0.1 --uicc0.imsi 001010000000001
```
**Expected Output**：

#### 6.2.1 gNB Connection Confirmation

> <img width="898" height="59" alt="image" src="https://github.com/user-attachments/assets/4629739f-117f-453f-ad29-7117171fcc8d" />
>
> <br>
> The base station successfully allocated the RNTI and established the RRC connection.

#### 6.2.2 nrUE Signaling Procedure

> <img width="1306" height="1205" alt="image" src="https://github.com/user-attachments/assets/beefdc4c-508a-4257-8faa-7dea86b90a68" />
>
> <br>
> Complete signaling flow on the UE side, covering physical layer synchronization through to NAS layer registration completion.

### 6.3 Data Plane Ping Testing
After successful IP allocation in the 5G Control Plane, the End-to-End (E2E) connectivity of the Data Plane must be verified. This section tests whether packets can successfully traverse the 5G Core Network (UPF) and reach the external data network (oai-ext-dn, IP: 192.168.70.135) via the UE's virtual interface `oaitun_ue1`.

```bash
# Basic connectivity test
ping -c 4 -I oaitun_ue1 192.168.70.135

# Stability and baseline benchmarking
ping -c 100 -I oaitun_ue1 192.168.70.135
```

**Expected Output**：

#### 6.3.1 Basic connectivity test

> <img width="1198" height="291" alt="image" src="https://github.com/user-attachments/assets/e5d1d4b6-8544-406d-a7f0-4aa802a018c9" />
>
> <br> 
> This step serves as the initial connectivity validation ("Hello World" test). The results show that all 4 ICMP packets were successfully returned with 0% packet loss and an average round-trip time (avg RTT) of approximately 10.1 ms. This confirms that the GTP-U tunnel between the UE, gNB, and 5G Core has been successfully established without routing obstructions.

#### 6.3.2 Stability and baseline benchmarking

><img width="1195" height="802" alt="image" src="https://github.com/user-attachments/assets/bf1c3c69-019b-4dff-8756-fae7df84161c" />
>
> <br>
> To ensure the accuracy of subsequent MAC layer scheduling analysis (such as UORA and BSR mechanisms), this step performs a stress and stability baseline test. Observing the continuous packet transmission data, the average latency remains stable at around 9.6 ms, with extremely low network jitter (mdev ~1.8 ms). The 0% loss rate and minimal jitter confirm that this OAI WSL2 virtual environment possesses high reliability and timing stability during prolonged operation.

---

## 7. Troubleshooting
If you encounter errors during deployment or execution, please refer to the table below for potential solutions:

| Symptoms | Root Cause | Solution |
| :--- | :--- | :--- |
| **gNB Crash on Startup**<br>`Assertion (b->th.nbAnt != 0) failed!` | The UE was not initialized with `ue.conf` or the `--sa` parameter was missing. | Copy the full nrUE command from Section 6.2; ensure both `-O .../ue.conf` and `--sa` are included. |
| **UE Fails to Obtain IP**<br>`Registration reject` | The IMSI used by the UE does not exist in the Core Network database. | Append `--uicc0.imsi 001010000000001` to the UE command to override the default ID. |
| **Connection Drop During Execution**<br>`Gap in writing to USRP`<br>`errno(14)` | WSL2 CPU scheduling jitter disrupted the timing-sensitive 5G physical layer. | Close resource-heavy background applications in Windows (e.g., browsers, antivirus) to free up CPU cycles. |
| **UPF Container Restart Loop**<br>`Exit Code 1` | The `develop` branch UPF now uses YAML; applying an old `.conf` format causes failure. | Check `docker-compose.yaml` and ensure the UPF mount path points to `/openair-upf/etc/config.yaml`. |
| **Ping `10.0.0.1` Failed**<br>`100% Packet Loss` | OAI UPF security rules drop ICMP packets destined for itself (False Alarm). | Perform a "Trans-network test" by pinging the external server directly (e.g., `192.168.70.135`). |
| **UE Sudden Crash**<br>`[CONFIG] unknown option: --sa` | Improper parameter ordering triggered a bug in the OAI argument parser. | Ensure the `--sa` parameter immediately follows the `-O .../ue.conf` argument. |


## 8. References
* [OAI 5G Core Network Deployment Tutorial](https://gitlab.eurecom.fr/oai/openairinterface5g/-/blob/develop/doc/NR_SA_Tutorial_OAI_CN5G.md)
* [OAI 5G SA nrUE Execution Tutorial](https://gitlab.eurecom.fr/oai/openairinterface5g/-/blob/develop/doc/NR_SA_Tutorial_OAI_nrUE.md?ref_type=heads)
