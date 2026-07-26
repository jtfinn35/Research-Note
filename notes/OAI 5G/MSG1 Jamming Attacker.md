## 4. Step-by-Step Guide

### 4.1 Workspace Setup and Source Code Cloning
To maintain a clean system directory, we will centralize all OAI projects within a dedicated workspace and clone the customized branch that supports the O1 interface and alert functionalities.

```bash
# Create and navigate to the unified workspace
mkdir ~/oai_workspace
cd ~/oai_workspace

# Clone the customized gNB repository and rename the folder to a clean 'OAI-gNB-O1'
git clone [https://github.com/bmw-ece-ntust/Richard-oai-gNB.git](https://github.com/bmw-ece-ntust/Richard-oai-gNB.git) OAI-gNB-O1
cd OAI-gNB-O1

# Switch to the specific branch supporting O1 telnet alerts
git switch O1-telnet-alert
```

### 4.2 Installing Dependencies
Before compiling, it is necessary to install the required system dependencies. By executing the build script with the `-I` parameter, the system will automatically handle all necessary environments, including the USRP drivers:

```bash
# Navigate to the build script directory
cd ~/oai_workspace/OAI-gNB-O1/cmake_targets/

# Install system and hardware-related dependencies
./build_oai -I
```

### 4.3 Build gNB and Telnet Module
Use Ninja for multi-core accelerated compilation, and ensure the Telnet Server (`telnetsrv`) shared library is built alongside it. This is a critical prerequisite for the O1 Adapter connection.

```bash
# Ensure you are still in the cmake_targets directory
sudo ./build_oai --ninja -c --gNB --nrUE --build-lib telnetsrv -w USRP -C    
```

> ⚠️ **Note on `sudo` usage:**
> Generally, do not use `sudo` with `./build_oai` as it may cause system permission errors. `sudo` is typically only required in Step 4.4 to access hardware interfaces. However, if your specific environment throws a permission error during compilation without it, you may append `sudo` to proceed.

**Core Parameter Description:**
*   `--ninja`: Uses `Ninja` instead of the traditional `make` to accelerate the build process.
*   `-c`: Cleans the previous build workspace.
*   `--gNB` / `--nrUE`: Specifies building the 5G Base Station (gNB) and User Equipment (nrUE) executables.
*   `--build-lib telnetsrv`: **[Critical]** Requests the compilation of an additional shared library for the Telnet Server, which is required for O1 Adapter integration.
*   `-w USRP`: Specifies the RF hardware equipment as `USRP`.
*   `-C`: Hard Clean; completely deletes and recreates the CMake build directory.

### 4.4 Run O1 Telnet Enabled Modem
Once the build is complete, navigate to the final `build` directory and start the softmodem using the specified configuration file. The startup command must explicitly enable the Telnet service and load the O1 module.

```bash
# Navigate to the executable directory
cd ~/oai_workspace/OAI-gNB-O1/cmake_targets/ran_build/build

# Start the gNB and enable the O1 Telnet function
sudo ./nr-softmodem -O ../../../targets/PROJECTS/GENERIC-NR-5GC/CONF/gnb.sa.band78.fr1.106PRB.usrpb210.conf --gNBs.[0].min_rxtxtime 6 -E --continuous-tx --log_config.PRACH_debug --telnetsrv --telnetsrv.shrmod o1
```

**Core Parameter Description:**
*   `-O [path]`: Specifies the path to the initialization configuration file required by the OAI gNB.
*   `--continuous-tx`: Configures the gNB to transmit signals continuously, which is critical for testing transmission stability.
*   `--log_config.PRACH_debug`: Enables debug logs related to PRACH (Physical Random Access Channel) for easier tracking.
*   `--telnetsrv`: Starts the Telnet Server, opening Port 9090 to allow external tools or the O1 Adapter to connect and control the modem.
*   `--telnetsrv.shrmod o1`: Loads the O1 shared module to officially interface with the O1 Adapter.
