.. _AM62D-signal-chain-from-linux-user-guide:

#############################
AM62D Signal Chain from Linux
#############################

********
Overview
********

This guide describes how to set up, build, and run the real-time signal chain parametric equalizer example by using the Texas Instruments AM62D audio evaluation module (EVM).
This demo example demonstrates real-time 3-stage parametric equalizer (biquad cascade) processing on C7x DSP with live audio input/output. The system processes continuous 8-channel audio streams at 48kHz sample rate through dedicated audio codecs, enabling real-time audio effects processing.
Below figure shows how this demo works:

.. figure:: /images/AM62D_Sigchain_Biquad_Demo.png
   :height: 500
   :width: 1100

**Initialization Sequence:**

- Step 1: CMA buffer setup request
   - A53 Linux application sends RPMsg request to C7x DSP with CMA buffer address for shared memory communication.

- Step 2: Message received by C7x DSP
   - C7x DSP receives RPMsg message containing shared memory buffer information via IPC framework.

- Step 3: Signal chain biquad application processes request
   - C7x signal chain application validates and configures the shared memory buffer for control/status communication.

- Step 4: Acknowledgment sent to Linux
   - C7x DSP sends acknowledgment message back to A53 Linux confirming successful setup completion.

- Step 5: RPMsg communication established
   - Bidirectional RPMsg communication channel is established between A53 Linux and C7x DSP.
   - Shared memory communication is configured using the CMA buffer for real-time control and monitoring data exchange.

- Step 6: Linux application receives confirmation
   - A53 Linux application receives setup confirmation indicating both RPMsg and shared memory are ready.
   - Linux application proceeds to initialize audio codecs and start demo with full communication established.

**Runtime Audio Processing:**

- Step 7: Real-time audio processing flow
   - Audio flows directly: **PCM6240 ADC → McASP → C7x L2 SRAM → Signal Processing → C7x L2 SRAM → McASP → TAD5212 DAC**
   - Audio data bypasses A53 Linux completely for optimal real-time performance.

   **Signal Chain Processing (within C7x DSP):**

   .. figure:: /images/AM62D_signal_chain_biquad_cascade.svg
      :height: 80
      :width: 800

   - Type Conversion (int32 → float): Converts multi-channel input data from int32_t to float format for processing.
   - Cascade Biquad Parametric EQ: 3-stage cascade biquad filter implementing parametric equalization using Direct Form I (DF1) biquad implementation.
   - Type Conversion (float → int32): Converts processed multi-channel output from float back to int32_t format.

**Parallel Control and Monitoring:**

- **Linux to C7x Control**: A53 Linux sends control commands (start/stop, coefficient updates) to C7x via shared memory buffers.

- **C7x to Linux Monitoring**: C7x DSP continuously updates performance metrics (DSP load, cycle count, throughput) in shared memory buffers.

- **C7x Control Processing**: C7x DSP reads control data from shared memory and applies configuration changes without interrupting audio processing.

- **GUI Monitoring Interface**: Host PC GUI utility connects to A53 Linux via Ethernet, retrieves monitoring data from shared memory, and displays real-time performance metrics and control interface.

**********************
Hardware Prerequisites
**********************

- `AM62D-EVM <https://www.ti.com/tool/AUDIO-AM62D-EVM>`__

- Audio device with input/output capability and 3.5mm audio jack and TRS compatible.

- SD card (minimum 16GB)

- USB Type-C 20W power supply (make sure to use type-C to type-C cable).

- USB-to-UART cable for console access

- PC (Windows or Linux) to flash image onto an SD Card

- Ethernet cable for signal chain examples host utility

- The ethernet expansion board `DP83867-EVM-AM <https://www.ti.com/tool/DP83867-EVM-AM>`__

- Host PC Requirements:

  - operating system:

    - Windows: |__WINDOWS_SUPPORTED_LONG__|

    - Ubuntu: |__LINUX_UBUNTU_VERSION_LONG__|

  - Memory: Minimum 4GB RAM (8GB or more recommended)

  - Storage: At least 10GB of free space

******************
Software and Tools
******************

- TI Processor SDK Linux RT (AM62Dx)

- MCU+ SDK for AM62Dx

- `C7000-CGT <https://www.ti.com/tool/C7000-CGT#downloads>`__ compiler

- `Code Composer Studio <https://software-dl.ti.com/mcu-plus-sdk/esd/AM62DX/11_00_00_16/exports/docs/api_guide_am62dx/CCS_PROJECTS_PAGE.html>`__

- `TI Clang Compiler Toolchain <https://www.ti.com/tool/download/ARM-CGT-CLANG>`__

- CMake, GCC, make, git, scp, minicom, Python3

- `rpmsg-dma library <https://github.com/TexasInstruments/rpmsg-dma/tree/main>`__

- Python 3 with tkinter, matplotlib, numpy for PC GUI utility

*********
EVM Setup
*********

#. Cable Connections

   - The figure below shows some important cable connections, ports and switches.

   - Take note of the location of the "BOOTMODE" switch for SD card boot mode.

        .. figure:: /images/AM62D_evm_setup.png
           :height: 600
           :width: 1000

   - Refer `Quick Start Guide for AM62D2-EVM <https://dev.ti.com/tirex/explore/node?isTheia=false&node=A__ARZXwcKmAzaWG2hcFPpKRA__PROCESSORS-DEVTOOLS__FUz-xrs__LATEST>`__ for more details.

#. Setup UART Terminal

   - First identify the UART port as enumerated on the host machine.

   - Make sure that the EVM and UART cable connected to  UART to USB port as shown in Cable Connections

   - In windows, you can use the "Device Manager" to see the detected UART ports
     - Search "Device Manager" in Windows Search Box in the Windows taskbar.

   - If you do not see any USB serial ports listed in "Device Manager" under "Ports (COM & LPT)", then make sure you have installed the UART to USB driver from `FTDI <https://www.ftdichip.com/drivers>`__.

   - For A53 Linux console select UART boot port (ex: COM34 in below screenshot), keep other options to default and set 115200 baud rate.

#. Setup SD card Boot Mode

   - EVM SD card  boot mode setting:

     - BOOTMODE [ 8 : 15 ] (SW3) = 0100 0000

     - BOOTMODE [ 0 : 7 ] (SW2) = 1100 0010

#. Audio Connections

   - Connect audio input source to EVM's MCASP input through connectors J1 and J2 (PCM6240 ADC).

   - Connect audio output to speakers/headphones via EVM's MCASP output through connectors J3 and J4 (TAD5212 DAC).

***********************************************
Steps to run real-time signal chain biquad demo
***********************************************

#. Flash an SD card with the :file:`tisdk-default-image-rt-am62dxx-evm.rootfs.wic.xz` image and follow the instructions provided at :ref:`Create SD Card <processor-sdk-linux-create-sd-card>` guide.

#. Insert the flashed SD card to `AUDIO-AM62D-EVM <https://www.ti.com/tool/AUDIO-AM62D-EVM>`__, connect the audio input/output, Ethernet Cable and power on TI AUDIO-AM62D-EVM.

#. Make sure the EVM boot mode switches position set for SD card boot as described earlier

#. Connect the USB-C cable from the power adapter to one of the two USB-C ports on the EVM.

#. Download Host Utility `rpmsg_sigchain_biquad_example_gui.py <https://github.com/TexasInstruments/rpmsg-dma/blob/scarthgap/example/sigchain_biquad/host_utility/rpmsg_sigchain_biquad_example_gui.py>`__.

#. The EVM should boot and the booting progress should display in the serial port console. Stop at u-boot prompt and enter following commands to apply :file:`k3-am62d2-evm-dsp-controlled-audio.dtbo` overlay. This device tree overlay disables Linux ALSA control of McASP2 and the audio codecs (PCM6240/TAD5212), allowing the C7x DSP to take direct hardware control of the audio interfaces.

   .. code-block:: console
      
      MMC:   mmc@fa10000: 0, mmc@fa00000: 1
      Loading Environment from nowhere... OK
      In:    serial@2800000
      Out:   serial@2800000
      Err:   serial@2800000
      Net:   eth0: ethernet@8000000port@1Could not get PHY for mdio@f00: addr 3
      Hit any key to stop autoboot: 0
      => setenv name_overlays ti/k3-am62d2-evm-dsp-controlled-audio.dtbo
      => boot 
   
#. At the end of booting, the Arago login prompt will appear. Just enter "root" to log in.

#. Get the EVM IP address

   .. code-block:: console

      root@am62dxx-evm:~# ifconfig

.. note::

   EVM IP address required for host utility to connect to demo application

#. Run signal chain biquad demo application from console

   There are two modes of operation:

   **GUI Mode (Network Server):**

   .. code-block:: console

      root@am62dxx-evm:~# rpmsg_sigchain_biquad_example

   **Command-Line Mode:**

   .. code-block:: console

      root@am62dxx-evm:~# rpmsg_sigchain_biquad_example start sleep:10 stop

#. For GUI Mode, on host machine launch the :file:`rpmsg_sigchain_biquad_example_gui.py` utility

   .. code-block:: console

      # python3 rpmsg_sigchain_biquad_example_gui.py <EVM IP address>
      Ex: # python3 rpmsg_sigchain_biquad_example_gui.py 192.168.0.101

#. The GUI utility automatically connects to the demo application and provides the following features:

   - Real-time visualization of:

     - C7x DSP load monitoring

     - Cycle count and throughput metrics

     - Processing status

   - Real-time biquad coefficient control

   - Demo start/stop control

   - Multi-port TCP communication:

     - Port 8888: Log messages from board to PC

     - Port 8889: Commands from PC to board (START/STOP)

     - Port 8890: Real-time C7x statistics streaming (JSON format)

#. For Command-Line Mode, the application provides:

   Available commands:

   - **start**: Start audio (auto-init codecs + start demo + display metrics)

   - **stop**: Stop audio (stop demo + auto-shutdown codecs + display final metrics)

   - **sleep:N**: Sleep for N seconds (with real-time performance monitoring)

   - **help**: Show usage information

   Examples:

   .. code-block:: console

      root@am62dxx-evm:~# rpmsg_sigchain_biquad_example start sleep:10 stop
      root@am62dxx-evm:~# rpmsg_sigchain_biquad_example start sleep:5 stop

   Return codes:

   - 0 = All commands executed successfully (PASSED)

   - 1 = Command sequence failed (FAILED)

.. note::

   Command-line mode provides automatic codec initialization/shutdown and real-time C7x performance metrics display during execution.

Below is sample snapshot of the GUI utility:

.. figure:: /images/AM62D_signal_chain_host_utility_snapshot.png
   :height: 800
   :width: 1200

- For more information on demo application and configuration, refer: `Signal Chain Biquad Example <https://github.com/TexasInstruments/rpmsg-dma/blob/main/example/sigchain_biquad/README.md>`__.

*************************************
How to build signal chain biquad demo
*************************************

Building signal chain biquad image from Yocto
=============================================

- To build the Signal Chain Biquad image, refer :ref:`Processor SDK - Building the SDK with Yocto <building-the-sdk-with-yocto>`

Building the Linux demo binary from sources
===========================================

#. The source code for Signal Chain Biquad demo is available as part of the `rpmsg-dma <https://github.com/TexasInstruments/rpmsg-dma/tree/main>`__.

   .. code-block:: console

      host# git clone https://github.com/TexasInstruments/rpmsg-dma.git -b main

#. Download and Install the AM62D Linux SDK from |__SDK_DOWNLOAD_URL__| following the steps mentioned at :ref:`Download and Install the SDK <download-and-install-sdk>`.

#. Prepare the environment for cross compilation.

   .. code-block:: console

      host# source <path to linux installer>/linux-devkit/environment-setup

#. Compile the source

    .. code-block:: console

       [linux-devkit]:> cd <path to rpmsg-dma source>
       [linux-devkit]:> cmake -S . -B build; cmake --build build

  - This will build:

    - The example application :file:`rpmsg_sigchain_biquad_example`

  - Transfer the generated files to SD card:

    - The example binary :file:`rpmsg_sigchain_biquad_example`  to :file:`/usr/bin`

    - The C7 DSP firmware file :file:`sigchain_biquad_cascade.c75ss0-0.release.strip.out` to :file:`/lib/firmware/`

  - Optional:

    - To build only the signal chain biquad example, use:

        .. code-block:: console

           cmake -S . -B build -DBUILD_SIGCHAIN_BIQUAD_EXAMPLE=ON

Building the C7x firmware from sources
======================================

#. Build the DSP Firmware using CCS projects or makefiles:

   **When using CCS projects to build**, import the CCS project from:
   :file:`examples/tisp/sigchain_dsp_rt/sigchain_biquad_cascade/am62dx-evm/c75ss0-0_freertos_linux/ti-c7000/`

   **When using makefiles to build**, build using make command:

   .. code-block:: console

      cd examples/tisp/sigchain_dsp_rt/sigchain_biquad_cascade/am62dx-evm/c75ss0-0_freertos_linux/ti-c7000/
      make clean && make

#. This generates :file:`sigchain_biquad_cascade.c75ss0-0.release.strip.out` firmware file

- Refer to the `MCU+ SDK Documentation  <https://software-dl.ti.com/mcu-plus-sdk/esd/AM62DX/11_02_00_20/exports/docs/api_guide_am62dx/GETTING_STARTED_BUILD.html>`__
- Refer to the `TISP Signal Chain Biquad Example <https://software-dl.ti.com/mcu-plus-sdk/esd/AM62DX/latest/exports/docs/api_guide_am62dx/EXAMPLES_TISP_SIGCHAIN_BIQUAD_LINUX_EXAMPLE.html>`__
