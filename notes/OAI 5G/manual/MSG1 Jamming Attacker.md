# MSG1 Jamming Attacker based on OAI Deployment Manual

![Version](https://img.shields.io/badge/version-v1.0-blue.svg)
![Status](https://img.shields.io/badge/status-verified-brightgreen.svg)
![OS](https://img.shields.io/badge/OS-Ubuntu_24.04-orange.svg)

> **Author**: 黃仁廷 (JTFinn)
> **Date Created**: 2026-07-26

> ⚠️ **Critical Safety Note**
> This manual outlines the procedure for configuring, building, and running the MSG1 Jamming Attacker based on the OpenAirInterface (OAI) UE. This tool is strictly designed for security testing and academic research. **It MUST be executed in a properly isolated RF environment** (e.g., using direct coaxial cable connections with attenuators, or inside an RF Shield Box) to prevent illegal interference with commercial 5G telecommunication networks.

---

## Table of Contents
## Table of Contents
- [1. Executive Summary](#1-executive-summary)
- [2. Architecture and Topology](#2-architecture-and-topology)
- [3. Prerequisites](#3-prerequisites)
- [4. Step-by-Step Guide](#4-step-by-step-guide)
  - [4.1 Workspace Setup and Building UHD Driver](#41-workspace-setup-and-building-uhd-driver)
  - [4.2 Clone MSG1 Attacker and Install Dependencies](#42-clone-msg1-attacker-and-install-dependencies)
  - [4.3 Build MSG1 Attacker](#43-build-msg1-attacker)
  - [4.4 Quick Re-build (After Code Edit)](#44-quick-re-build-after-code-edit)
  - [4.5 Execute Attacker](#45-execute-attacker)
- [5. Configuration and Parameters](#5-configuration-and-parameters)
  - [5.1 Core Execution Parameters](#51-core-execution-parameters)
  - [5.2 Advanced Attack Options](#52-advanced-attack-options)
- [6. Verification and Safety Guidelines](#6-verification-and-safety-guidelines)
- [7. Troubleshooting](#7-troubleshooting)
- [8. References](#8-references)

---

## 1. Executive Summary
This document provides a standardized operating procedure for compiling and executing the MSG1 Jamming Attacker. By modifying the OpenAirInterface (OAI) User Equipment (UE) source code, this tool leverages a Software Defined Radio (SDR) device to continuously transmit MSG1 (PRACH Preamble) signals towards a target 5G Base Station (gNB). This capability is primarily used to evaluate gNB resilience against Random Access Channel (RACH) flooding or synchronization jamming attacks in a controlled testing environment.

---

## 2. Architecture and Topology
The system architecture isolates the attacker node from the target gNB. The attacker host uses the customized OAI UE softmodem to command the SDR via the UHD driver, sending jamming signals over the Uu interface.

```mermaid
graph TD
    %% Define Styles
    classDef host fill:#2ca02c,stroke:#1b5e20,stroke-width:2px,color:#fff;
    classDef proc fill:#1f77b4,stroke:#0d47a1,stroke-width:2px,color:#fff;
    classDef sdr fill:#ff7f0e,stroke:#e65100,stroke-width:2px,color:#fff;
    classDef target fill:#d50000,stroke:#b71c1c,stroke-width:2px,color:#fff;

    subgraph Attacker_Node ["Attacker Node (Ubuntu Host)"]
        Host["Ubuntu 24.04 LTS Host"]:::host
        Process["OAI UE Attacker Process<br/>(nr-uesoftmodem)"]:::proc
        UHD["UHD Driver (SDR Interface)"]:::proc
        
        Host --- Process
        Process --- UHD
    end

    USRP["USRP SDR Device<br/>(B210 / N300 / X300)"]:::sdr
    gNB["OAI gNB<br/>(Target Base Station)"]:::target

    UHD --- |"USB / PCIe / Ethernet"| USRP
    USRP --- |"RF Jamming (Uu Interface)"| gNB
```

---

## 3. Prerequisites
Ensure the hardware, operating system, and safety environments meet the minimum requirements before proceeding with the build process.

*   **Host Machine**: x86_64 architecture CPU (Minimum 8 cores @ 3.5 GHz), 8 GB RAM.
    *   *Note: It is highly recommended to run the attacker on a separate, dedicated host from the target gNB.*
*   **Operating System**: Ubuntu 24.04 LTS (Native installation is recommended to ensure stable USB/PCIe passthrough for the SDR).
*   **Supported SDR Hardware**: USRP B210, USRP N300, or USRP X300.
    *   Identify the network interface(s) or USB ports where the USRP is connected and ensure sufficient power supply.
*   **RF Safety Equipment**: Coaxial cables paired with appropriate RF attenuators, or a Faraday cage, are mandatory for execution. Do not transmit over-the-air (OTA) without isolation.

---

## 4. Step-by-Step Guide

### 4.1 Workspace Setup and Building UHD Driver
To maintain a clean system directory, we will centralize the SDR drivers and the attacker project within a dedicated workspace. Before compiling the attacker, the Universal Hardware Driver (UHD) must be built from the source.

```bash
# Create and navigate to the unified workspace
mkdir ~/oai_workspace
cd ~/oai_workspace

# Install system and hardware-related dependencies
sudo apt install -y autoconf automake build-essential ccache cmake cpufrequtils doxygen ethtool g++ git inetutils-tools libboost-all-dev libncurses-dev libusb-1.0-0 libusb-1.0-0-dev libusb-dev python3-dev python3-mako python3-numpy python3-requests python3-scipy python3-setuptools python3-ruamel.yaml

# Clone UHD repository and checkout version 4.8.0.0
git clone [https://github.com/EttusResearch/uhd.git](https://github.com/EttusResearch/uhd.git)
cd uhd
git checkout v4.8.0.0

# Build and install UHD
cd host
mkdir build
cd build
cmake ../
make -j $(nproc)
sudo make install
sudo ldconfig

# Download USRP FPGA images
sudo uhd_images_downloader
```

### 4.2 Clone MSG1 Attacker and Install Dependencies
Return to the workspace to clone the customized OAI UE repository. Ensure you use a GitHub Personal Access Token (PAT) for authentication when prompted for a password.

```bash
cd ~/oai_workspace

# Clone the customized MSG1 attacker repository
git clone [https://github.com/bmw-ece-ntust/Richard-OAI-UE-MSG1-attacker.git](https://github.com/bmw-ece-ntust/Richard-OAI-UE-MSG1-attacker.git) OAI-UE-MSG1-attacker
cd OAI-UE-MSG1-attacker
git checkout 2026-w26

# Install OAI-specific dependencies
cd cmake_targets
./build_oai -I

# Install nrscope dependencies
sudo apt install -y libforms-dev libforms-bin
```

### 4.3 Build MSG1 Attacker
Initiate the compilation of the attacker executable along with the required libraries.

```bash
# Ensure you are in the cmake_targets directory
cd ~/oai_workspace/OAI-UE-MSG1-attacker/cmake_targets

# Build the MSG1 attacker
./build_oai -w USRP --ninja --nrUE --gNB --build-lib "nrscope" -C
```

### 4.4 Quick Re-build (After Code Edit)
If you modify the C/C++ source code to alter jamming logic or parameters, you do not need to run the full `build_oai` script again. Use the following command for a rapid incremental build.

```bash
cd ~/oai_workspace/OAI-UE-MSG1-attacker/cmake_targets/ran_build/build
sudo ninja nr-softmodem nr-uesoftmodem dfts ldpc params_libconfig
```

### 4.5 Execute Attacker
Once the build is complete, navigate to the execution directory to launch the attacker. **Ensure your hardware is physically connected via cables and attenuators before executing this command.**

```bash
cd ~/oai_workspace/OAI-UE-MSG1-attacker/cmake_targets/ran_build/build

# Launch the UE softmodem to broadcast MSG1 jamming signals
sudo ./nr-uesoftmodem -r 106 --numerology 1 --band 78 -C 3619200000 --ssb 516 -E --ue-fo-compensation
```

---

## 5. Configuration and Parameters
The MSG1 Jamming Attacker uses the OAI `nr-uesoftmodem` executable. Proper configuration of RF parameters is required to synchronize with the target gNB. Below are the key parameters used during execution:

### 5.1 Core Execution Parameters
*   `-r <value>`: Number of Resource Blocks (RB). Determines the bandwidth of the transmitted signal (e.g., `106` for 40MHz).
*   `--numerology <value>`: Subcarrier Spacing (SCS). A value of `1` indicates 30 kHz spacing (typical for 5G NR FR1).
*   `--band <value>`: The specific 5G frequency band to operate on (e.g., `78` for n78 / 3.3-3.8 GHz).
*   `-C <frequency>`: The center frequency in Hz. This must strictly match the target gNB's SSB ARFCN converted to Hz (e.g., `3619200000` for ~3.6 GHz).
*   `--ssb <offset>`: The Synchronization Signal Block (SSB) index. The attacker must align with this to transmit the MSG1 precisely.
*   `-E`: Enables External Band Filter or specific hardware enhancements for the RF front-end.
*   `--ue-fo-compensation`: Enables automatic frequency offset compensation to prevent drift caused by hardware thermal issues.

### 5.2 Advanced Attack Options

> 💡 **Parameter Compatibility Warning**
> The standard OAI UE arguments for adjusting Tx/Rx gains are normally `--tx_gain` and `--rx_gain`. The parameters `-ue-txgain` and `-ue-rxgain` listed below may be customized specifically for the `Richard-OAI-UE-MSG1-attacker` branch. If you encounter an `unrecognized option` error during execution, please revert to the standard OAI parameters.

*   `--seq <value>`: Specifies the number of preambles to include in one root sequence to increase collision probability.
*   `--ue-max-power <value>`: Overrides the UE maximum transmission power limitation.
*   `-ue-txgain <value>`: Adjusts the UE transmit power (Tx Gain). Higher values (e.g., `120`) increase jamming intensity but risk hardware overheating.
*   `-ue-rxgain <value>`: Adjusts the UE receive gain (Rx Gain).

---

## 6. Verification and Safety Guidelines
To verify the execution of the attacker, you must monitor the terminal output for synchronization logs with the gNB. However, **strict RF isolation protocols must be followed** before initiating the transmission.

> 🛑 **CRITICAL SAFETY WARNING: DO NOT TRANSMIT OVER-THE-AIR**
> Currently, the target gNB does not have a USRP connected, meaning a closed-loop, cabled RF testing environment has not been established. **Under no circumstances should you execute the attacker command with an antenna attached to transmit signals into the open air (OTA).**
>
> The configured frequency (n78 band / 3.6 GHz) belongs to licensed commercial 5G spectrum. Transmitting MSG1 jamming signals without a coaxial cable + attenuator setup or a Faraday cage will directly disrupt surrounding public telecommunication networks, violating federal telecom regulations. Wait until both the gNB and Attacker SDRs are physically connected via RF cables before running the execution command.

---

## 7. Troubleshooting

| Symptoms | Root Cause | Solution |
| :--- | :--- | :--- |
| **GitHub Authentication Failed**<br>`remote: Invalid username or token.`<br>`Password authentication is not supported...` | GitHub deprecated account password authentication for command-line Git operations in 2021. | Go to GitHub web **Settings** > **Developer settings** > **Personal access tokens (classic)**. Generate a new token with the `repo` scope checked. When prompted for a password during `git clone`, paste this token (starts with `ghp_`) instead of your account password. |
| **Directory Not Found Cascade Error**<br>`cd: OAI-UE-MSG1-attacker: No such file or directory`<br>`./build_oai: No such file or directory` | This is a cascading failure caused by bulk copy-pasting commands. The `git clone` command failed (due to the auth issue above), so the folder was never created, causing all subsequent commands to fail. | Resolve the GitHub token issue first. Always execute the `git clone` command individually. Verify the repository has been downloaded 100% successfully before running the subsequent `cd` and `build` commands line-by-line. |
| **UHD Device Not Found**<br>`No UHD devices found with address...` | The system cannot detect the USRP B210/X300 device via USB or Ethernet. | Ensure the USRP is properly powered and connected. If using a VM or WSL2, verify that USB/PCIe passthrough is actively forwarding the device to the Linux kernel. Check visibility by running the `uhd_find_devices` command. |

---

## 8. References
* [NTUST Customized MSG1 Attacker Repository (Richard-OAI-UE-MSG1-attacker)](https://github.com/bmw-ece-ntust/Richard-OAI-UE-MSG1-attacker)
* [OpenAirInterface (OAI) Official GitLab Repository](https://gitlab.eurecom.fr/oai/openairinterface5g)
* [OAI NR UE Official Execution Tutorial (COTS UE Setup)](https://gitlab.eurecom.fr/oai/openairinterface5g/-/blob/develop/doc/NR_SA_Tutorial_COTS_UE.md)
* [Ettus Research UHD (USRP Hardware Driver) Build and Install Guide](https://files.ettus.com/manual/page_build_guide.html)
* [USRP B210 Hardware Specifications (Tx Power Limits)](https://kb.ettus.com/B200/B210/B200mini/B205mini)
* [OAI 5G NR Execution and Parameters Documentation (RUNMODEM.md)](https://gitlab.eurecom.fr/oai/openairinterface5g/-/blob/develop/doc/RUNMODEM.md)
* [3GPP TS 38.211: NR; Physical channels and modulation (PRACH Specifications)](https://portal.3gpp.org/desktopmodules/Specifications/SpecificationDetails.aspx?specificationId=3213)
* [NCC Taiwan Frequency Allocation Table (Safety & Regulations)](https://www.ncc.gov.tw/chinese/show_file.aspx?table_name=news&file_sn=43407)
* [GitHub Docs: Managing your personal access tokens (Classic)](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens)
