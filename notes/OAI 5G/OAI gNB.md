# OAI gNB with O1 Interface (Telnet Enabled) Deployment Manual

![Version](https://img.shields.io/badge/version-v1.0-blue.svg)
![Status](https://img.shields.io/badge/status-verified-brightgreen.svg)
![OS](https://img.shields.io/badge/OS-Ubuntu_22.04-orange.svg)

> **Author**: 黃仁廷 (JTFinn)
> **Date Created**: 2026-07-26

> ⚠️ **Note**
> This manual outlines the procedure for building and running the OpenAirInterface (OAI) gNB with the `telnetsrv` shared library enabled. This specific configuration is a strict prerequisite for enabling the OAI O1 Adapter to function properly and interface with the SMO Management Layer. 

---

## 1. Executive Summary
This document provides a standardized operating procedure for compiling, configuring, and executing the OpenAirInterface (OAI) 5G Base Station (gNB) with O1 management capabilities. By compiling a customized branch and enabling the telnet server module (`telnetsrv`), the gNB exposes a control interface on port 9090. This allows the standalone O1 Adapter to connect, enabling critical management functions such as Performance Management (PM), Configuration Management (CM), and Fault Management (FM) via NETCONF and VES protocols. 

---

## 2. Architecture and Topology
The system architecture integrates the traditional OAI RAN components with the SMO (Service Management and Orchestration) layer through the O1 interface. The topology is structured as follows:

```mermaid
graph TD
    %% Define Styles
    classDef host fill:#2ca02c,stroke:#1b5e20,stroke-width:2px,color:#fff;
    classDef proc fill:#1f77b4,stroke:#0d47a1,stroke-width:2px,color:#fff;
    classDef sdr fill:#ff7f0e,stroke:#e65100,stroke-width:2px,color:#fff;
    classDef smo fill:#9467bd,stroke:#4a148c,stroke-width:2px,color:#fff;

    subgraph gNB_Host ["gNB Host (Ubuntu)"]
        gNB_Process["OAI gNB Process<br/>(nr-softmodem)"]:::proc
        TelnetSrv["telnetsrv Shared Lib<br/>(Port 9090)"]:::proc
        O1_Adpt["OAI O1 Adapter"]:::proc
        gNB_Process --- TelnetSrv
    end

    subgraph SMO_Layer ["SMO Management Layer"]
        VES["VES Collector"]:::smo
        NETCONF["NETCONF Server"]:::smo
    end

    RU["Radio Unit"]:::sdr

    %% Connections
    gNB_Process --- |"(Fronthaul)"| RU
    O1_Adpt --- |"Telnet Commands (Port 9090)"| TelnetSrv
    O1_Adpt -->|"VES Events (O1)"| VES
    O1_Adpt <-->|"NETCONF (O1)"| NETCONF
```

---

## 3. Prerequisites
The compilation and execution environment remains identical to the standard OAI 5G SA deployment to ensure consistency and prevent Out of Memory (OOM) crashes during massive C++ parallel builds.

*   **Host Machine**: Acer Predator PHN16-72 (32GB RAM, 32 Logical Cores).
*   **Virtualization Environment**: Windows Subsystem for Linux (WSL2) v2.6.3.
    *   **Performance Tuning Strategy**: The Windows `%userprofile%\.wslconfig` file is explicitly configured to allocate sufficient resources:
        ```ini
        [wsl2]
        memory=24GB
        processors=24
        swap=8GB
        ```
*   **Operating System**: Ubuntu 22.04 LTS (Kernel 6.6.87).

---

## 4. Step-by-Step Guide

### 4.1 Installing Dependencies
Before compiling OAI, it is necessary to install the required system dependencies. By executing the build script with the `-I` parameter, the system will automatically handle all necessary environments, including the USRP drivers:

```bash
cd openairinterface5g/cmake_targets/
./build_oai -I
```

### 4.2 Clone Customized Repository and Build
To support the O1 interface and specific alert functionalities, we do not use the official main branch. Instead, clone the customized `Richard-oai-gNB` repository, switch to the specific branch, and then perform a parallel build.

```bash
git clone https://github.com/bmw-ece-ntust/Richard-oai-gNB.git
git switch O1-telnet-alert
cd openairinterface5g/cmake_targets/
sudo ./build_oai --ninja -c --gNB --nrUE --build-lib telnetsrv -w USRP -C    
```

:warning: Note on sudo usage:
Generally, do not use sudo with ./build_oai as it may cause system permission errors. sudo is typically only required in Step 4.3 to access hardware interfaces. However, if your specific environment throws a permission error without it, you may append sudo to proceed.

**Parameter Description:**
*   `sudo`: Executes the command with superuser (root) privileges.
*   `--ninja`: Uses `Ninja` instead of the traditional `make` to accelerate the build process.
*   `-c`: Cleans the previous build workspace.
*   `--gNB`: Specifies building the 5G Base Station (gNB) executable, which generates the `nr-softmodem` program.
*   `--nrUE`: Specifies building the 5G User Equipment (nrUE) executable, which generates the `nr-uesoftmodem` program.
*   `--build-lib telnetsrv`: Requests the compilation of an additional shared library for the **Telnet Server**, which is crucial for the O1 Adapter connection.
*   `-w USRP`: Specifies the RF hardware equipment as `USRP`.
*   `-C`: Hard Clean; completely deletes and recreates the CMake build directory.

### 4.3 Run Modem (O1 Telnet Enabled)
Navigate to the build directory and start the softmodem using the specified configuration file. The startup command must explicitly enable the Telnet service and the continuous transmission mode.

```bash
cd openairinterface5g/cmake_targets/ran_build/build

sudo ./nr-softmodem -O ../../../targets/PROJECTS/GENERIC-NR-5GC/CONF/gnb.sa.band78.fr1.106PRB.usrpb210.conf --gNBs.[0].min_rxtxtime 6 -E --continuous-tx --log_config.PRACH_debug --telnetsrv --telnetsrv.shrmod o1
```

**Parameter Description:**
*   `-O [path]`: Specifies the path to the initialization configuration file required by the OAI gNB.
*   `--gNBs.[0].min_rxtxtime 6`: Sets the minimum RX-to-TX time for Node 0 to 6 slots.
*   `-E`: Enables certain additional execution modes.
*   `--continuous-tx`: Configures the gNB to transmit signals continuously, which is critical for testing transmission stability.
*   `--log_config.PRACH_debug`: Enables debug logs related to PRACH (Physical Random Access Channel) for easier tracking.
*   `--telnetsrv`: Starts the Telnet Server, allowing external tools or the O1 Adapter to connect and control the modem.
*   `--telnetsrv.shrmod o1`: Loads the O1 shared module to officially interface with the O1 Adapter.

---

## 5. Configuration
No special configuration modifications are required specifically for the O1 telnet module. The system utilizes the default USRP B210 Standalone configuration file (`gnb.sa.band78.fr1.106PRB.usrpb210.conf`). 

---

## 6. Verification
To verify that the gNB has successfully started and the O1 Telnet management channel is operational, you can use the following commands in a new terminal window:

*   **Check if the gNB process is running in the background:**
    ```bash
    ps aux | grep nr-softmodem
    ```
*   **Verify the Telnet Server is actively listening on Port 9090:**
    ```bash
    ss -tulpn | grep 9090
    ```
*   **Test local connectivity to the Telnet interface:**
    ```bash
    telnet 127.0.0.1 9090
    ```

---

## 7. Troubleshooting

> **Critical Note regarding Verification:** 
> The verification tests in Section 6 require the gNB process to be running continuously. However, if you execute the startup command with the `-w USRP` parameter but do not have physical USRP hardware connected, the gNB will immediately crash upon startup with a `No USRP Device Found` error. Consequently, the program will terminate, and you will not be able to successfully perform the port listening or telnet connection tests. To resolve this and proceed with testing, you must either connect a physical USRP device or switch the execution mode to the RF Simulator (`--rfsim`).

| Symptoms | Root Cause | Solution |
| :--- | :--- | :--- |
| **No USRP Device Found**<br>`can't open the radio device: none`<br>`Assertion (0) failed!` | The `-w USRP` parameter was used during build/execution, but no physical USRP SDR is connected to the host. | Connect a USRP via USB 3.0 and ensure passthrough to WSL2 is active. Alternatively, recompile with RF Simulator enabled and replace `-w USRP` with `--rfsim` in your execution command. |
| **SCTP Socket Bind Error**<br>`SCTP_BINDX_ADD_ADDR failed: errno 99`<br>`Cannot assign requested address` | The gNB configuration file attempts to bind to an IP address (e.g., `192.168.70.129`) that does not exist on the host's network interfaces. | Run `ip a` to check your WSL2 host IP. Open the `gnb.sa.band78...usrpb210.conf` file and update the IP addresses in the `NETWORK_INTERFACES` section to match your actual host IP. |
| **Command Not Found**<br>`sudo: ./nr-softmodem: command not found` | The terminal is in the incorrect directory (e.g., `cmake_targets`) and cannot locate the built executable. | Ensure you navigate into the final build output directory using `cd ~/openairinterface5g/cmake_targets/ran_build/build` before executing the modem. |

---

## 8. References
* [OAI Official Documentation](https://gitlab.eurecom.fr/oai/openairinterface5g)
* [O-RAN O1 Interface Specifications](https://www.o-ran.org/specifications)
