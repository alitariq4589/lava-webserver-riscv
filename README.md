# Setting up RISC-V board with LAVA for KernelCI

This document describes how one can integrate Banana Pi F3 (and a Raspberry pi device) with Linaro's Automation Validation Architecture (LAVA) webserver.

Documentation chronological sequence is as follows.

## [Getting Started](/docs/GETTING_STARTED.md): 

- Describes how to install LAVA webserver
- Describes the difference between Worker, Master/server, device-types, devices
- Describes how to add workers, device-types, devices
- Describes how to add the QEMU device as the first device
- Describes how to run the first job on QEMU device

## [Adding first physical device (Raspberry Pi 4 Model B)](/docs/ADDING_RPI_4B.md):

- Describes how to add first physical device (Raspberry Pi 4 Model B) in LAVA
- Describes how to setup PDU even if you dont have a PDU with Arduino and Raspberry Pi 4 Model B
- Describes how to add device template and connection configuration in `/etc/ser2net.yaml`
- Describes how to run first job to check the connection

## [Setting up RISC-V device (Banana Pi F3) for KernelCI](/docs/SETUP_BANANAPIF3.md):

- Describes the bootflow of Banana Pi F3
- Describes how to format the sd card and prepare the Board for KernelCI/LAVA
- Describes where to get the binary files (FSBL, U-Boot etc.)

## [Adding RISC-V device (Banana Pi F3)](/docs/ADDING_BPI-F3.md):

- Describes how to add a newer physical board in LAVA worker and then add it in the lava-server
- Describes how to set up PDU with arduino uno and relays 

## [LAVA Web-Server for RISCV boot and deployment](/docs/Lava-bpi-f3-bootflow.md):
- Setting up Banana Pi F3 Boot Flow for Kernel CI
  - U-Boot Secondary Program Loader (SPL) Setup
  - U-Boot environment for BPI-F3
  - Host Machine NFS Server Setup
  - Booting the Linux kernel on BPI-F3
- Automating with LAVA Job (.yaml) File
