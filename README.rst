=====================================================
OpenAirInterface (OAI) 5G Setup Cookbook
=====================================================

.. admonition:: Overview
   :class: note

   This cookbook assumes you are running the exact device configurations listed in the :ref:`Devices Info <Devices_Info>` section. It does not include initial installation or optimization steps.

   **Do not use this guide to set up systems with different hardware/software configurations.**

   This guide assumes the PLMN ``00101`` is used throughout the deployment.


Cold Start
==========

1. OS Preparation
-----------------

1. Reboot the PC. When the GRUB menu appears, select ``Advanced options for Ubuntu``, then choose ``Ubuntu, with Linux 6.8.0-90-lowlatency``.

2. Verify that Ubuntu is using the low-latency kernel:

   .. code-block:: bash

       uname -r

   .. code-block:: console

       6.8.0-90-lowlatency

3. Verify that the current user has the privileges required to set real-time process priority up to 99:

   .. code-block:: bash
       
       ulimit -r

   .. code-block:: console

       99

2. SDR (USRP B210) Preparation
------------------------------

1. Mount two antennas to the **RF-A** ``TX/RX`` and ``RX2`` ports.
2. Connect the 12V power supply to the B210.
3. Connect the B210 to a USB 3.0 port on the computer using a USB 3.0 cable. USB 3.0 ports are typically blue or marked with "USB 3.0" or the "SS" (SuperSpeed) symbol.
4. Verify the connection by running the following command in the terminal and checking the highlighted output:

   .. code-block:: bash
       
       sudo uhd_find_devices   

   .. code-block:: console
       :emphasize-lines: 8

       [INFO] [UHD] linux; GNU C++ version 13.3.0; Boost_108300; UHD_4.8.0.HEAD-0-g308126a4
       --------------------------------------------------
       -- UHD Device 0
       --------------------------------------------------
       Device Address:
           serial: <YOUR_SERIAL_HERE>
           name: <YOUR_DEVICE_NAME>
           product: B210
           type: b200
    
   .. code-block:: bash
    
       lsusb -t

   .. code-block:: console
       :emphasize-lines: 8-12

       /:  Bus 001.Port 001: Dev 001, Class=root_hub, Driver=xhci_hcd/16p, 480M
           |__ Port 008: Dev 002, If 0, Class=Wireless, Driver=btusb, 12M
           |__ Port 008: Dev 002, If 1, Class=Wireless, Driver=btusb, 12M
           |__ Port 013: Dev 003, If 0, Class=Human Interface Device, Driver=usbhid, 1.5M
       /:  Bus 002.Port 001: Dev 001, Class=root_hub, Driver=xhci_hcd/10p, 5000M
       /:  Bus 003.Port 001: Dev 001, Class=root_hub, Driver=xhci_hcd/2p, 480M
       /:  Bus 004.Port 001: Dev 001, Class=root_hub, Driver=xhci_hcd/2p, 10000M
           |__ Port 001: Dev 002, If 0, Class=Vendor Specific Class, Driver=[none], 5000M
           |__ Port 001: Dev 002, If 1, Class=Vendor Specific Class, Driver=[none], 5000M
           |__ Port 001: Dev 002, If 2, Class=Vendor Specific Class, Driver=[none], 5000M
           |__ Port 001: Dev 002, If 3, Class=Vendor Specific Class, Driver=[none], 5000M
           |__ Port 001: Dev 002, If 4, Class=Vendor Specific Class, Driver=[none], 5000M

   .. admonition:: Note
       :class: note

       Ensure the USB link speed is ``5000M``, which indicates a successful USB 3.0 connection. If you see ``480M``, try reconnecting the B210 to a different port.

3. CPU/Fan Preparation
----------------------

1. Set the fans to maximum speed to prevent thermal throttling from interrupting OAI processes.

   .. code-block:: bash

       sudo systemctl stop i8kmon
       i8kctl fan 2 2 

2. Disable CPU idle states, set the CPU to performance mode, and lock the frequency to 4.4 GHz.

   .. code-block:: bash

       sudo cpupower idle-set -D 0
       for ((i=0;i<$(nproc);i++)); do sudo cpufreq-set -c $i -r -g performance; done
       sudo cpupower frequency-set -g performance
       sudo cpupower frequency-set --min 4.40GHz --max 4.40GHz --governor performance

   .. admonition:: Note
       :class: note

       The i7-9700K supports an all-core max boost frequency of 4.6 GHz. However, extensive testing on the XPS 8930 indicates that locking the frequency above 4.4 GHz can cause instability under high load, even with fans at maximum speed.

3. Verify the CPU frequency and fan speed. You should hear the fans running at full speed. Monitor them in real-time:

   .. code-block:: bash

       watch -n 1 sensors
       watch -n 1 "grep MHz /proc/cpuinfo"

4. OAI-CN (Core Network) Setup
------------------------------

1. Start the CN services:

   .. code-block:: bash

       cd ~/oai-cn5g
       docker compose up -d

   .. admonition:: Note
       :class: note

       Verify container health and logs to ensure the CN is running properly before proceeding to the RAN setup.

       .. code-block:: bash

           docker compose ps 
           docker compose logs -f

       All containers (e.g., ``oai-amf``, ``oai-smf``, ``oai-upf``) should show a ``(healthy)`` status. Resolve any unhealthy containers before proceeding.

5. OAI-RAN (Radio Access Network) Setup
---------------------------------------

1. Start the RAN services (keep this terminal session open):

   .. code-block:: bash

       cd ~/openairinterface5g/cmake_targets/ran_build/build
       sudo ./nr-softmodem -O ../../../targets/PROJECTS/GENERIC-NR-5GC/CONF/gnb.sa.band78.fr1.106PRB.usrpb210.conf -E --continuous-tx

2. Verify console output:
   
   * The gNB is running successfully if you see standard framing logs:
     
     .. code-block:: console

         [NR_MAC] Frame.Slot 768.0
         [NR_MAC] Frame.Slot 896.0

   * The UE has successfully connected if you see the Random Access (RA) procedure and RRC connection logs:
     
     .. code-block:: console

         [NR_MAC] 91.19 UE RA-RNTI 010b TC-RNTI 584f: initiating RA procedure
         [NR_RRC] [UL] (cellID bc614e, UE ID 1 RNTI 584f) Received RRCSetupComplete (RRC_CONNECTED reached)
         [NGAP]   Create UE context (ID 1) for AMF 'OAI-AMF' (assoc_id 5)

   * You should **NOT** see continuous red errors or repeated ``UUULLLLLLLLL`` output (indicating hardware underrun/overrun).

6. RM520N-GL Setup
------------------

Mount antennas to all four ports using the same antenna type (the included GPS antenna is not required).

The RM520N-GL runs automatically without additional setup. Once powered, it will attempt to connect to the 5G network.


Stop
====

1. Stop the CN
--------------

.. code-block:: bash

    cd ~/oai-cn5g
    docker compose down -v

2. Stop the RAN
---------------

Terminate the softmodem process by pressing ``Ctrl+C`` in the terminal.

3. (Optional) Restore CPU/Fan States
------------------------------------

The easiest way to restore system settings is to reboot. To restore manually:

.. code-block:: bash

    # Restore CPU control
    sudo cpupower idle-set -E
    sudo cpupower frequency-set --min 800MHz --max 4.60GHz --governor powersave

    # Restore Fan control
    i8kctl fan 1 1
    sudo systemctl start i8kmon


.. _Restart:

Restart
=======

1. Turn off the UE (module).
2. Restart the CN:

   .. code-block:: bash

       cd ~/oai-cn5g
       docker compose down -v
       docker compose up -d

3. Restart the RAN (terminate via ``Ctrl+C``, then relaunch):

   .. code-block:: bash

       cd ~/openairinterface5g/cmake_targets/ran_build/build
       sudo ./nr-softmodem -O ../../../targets/PROJECTS/GENERIC-NR-5GC/CONF/gnb.sa.band78.fr1.106PRB.usrpb210.conf -E --continuous-tx


Add User (Flash SIM Card)
=========================

1. Prepare the SIM Card Reader
------------------------------

Plug the HID OMNIKEY 3121 SIM card reader into the PC. Verify access to the PC/SC reader without ``sudo``:

.. code-block:: bash

    pcsc_scan

.. code-block:: console
    :emphasize-lines: 5

    PC/SC device scanner
    Scanning present readers...
    0: HID Global OMNIKEY 3x21 Smart Card Reader [OMNIKEY 3x21 Smart Card Reader] 00 00

2. Prepare the New User Info
----------------------------

Each user in the 5G network must have a unique IMSI. 

* Must be pure numeric, 15 digits.
* Starts with ``00101`` (per our PLMN).
* Must not exist in the current database.

Example: ``001010100016069``

Check existing IMSIs:

.. code-block:: bash

    grep "00101" /home/i3t/oai-cn5g/database/oai_db.sql

3. Flash the SIM Card
---------------------

Look up your sysmoISIM-SJA5-9FV's ADM key using the **last row of digits** printed on the nano SIM. This matches the ICCID.

.. admonition:: Warning
    :class: warning

    Do **NOT** use the IMSI printed on the external SIM card holder to look up the ADM key. Make sure you use the correct ADM key; three failed attempts will **permanently** lock the SIM card!

Replace ``<ADM>``, ``<IMSI>``, ``<KEY>``, and ``<OPC>`` in the command below:
                
.. code-block:: bash

    conda activate sim_card
    cd ~/sim_card/pysim
    ./pySim-prog.py -p 0 -t sysmoISIM-SJA5 -a <ADM> -x 001 -y 01 -i <IMSI> -s 8988211000000110000 -o <OPC> -k <KEY>


.. list-table:: SIM Card Identifiers
   :header-rows: 1
   :widths: 15 40 20 25

   * - Term
     - Description
     - Modifiable?
     - Notes
   * - ICCID
     - Integrated Circuit Card ID
     - No
     - Printed on the **nano** SIM. Used to lookup ADM key.
   * - ADM
     - Administrative Key
     - No
     - Unique per SIM. 3 wrong attempts locks the card.
   * - IMSI
     - International Mobile Subscriber Identity
     - Yes
     - Unique account ID. Must be 15 digits (e.g., starts with 00101).
   * - KEY (K)
     - Secret Key
     - Yes
     - Used for authentication. Must match the CN database.
   * - OPC
     - Operator Key
     - Yes
     - Derived from the OP and KEY. Must match the CN database.
     
4. Insert the New User into the CN Database
-------------------------------------------

1. **User Authentication Data:** Replace values in the following SQL snippet and insert it into ``~/oai-cn5g/database/oai_db.sql`` under the ``AuthenticationSubscription`` table:

   .. code-block:: sql

       ('<IMSI>', '5G_AKA', '<KEY>', '<KEY>', '{"sqn": "000000000000", "sqnScheme": "NON_TIME_BASED", "lastIndexes": {"ausf": 0}}', '8000', 'milenage', '<OPC>', NULL, NULL, NULL, NULL, '<IMSI>');

2. **User Session Management Data:** Replace the IP address with a unique IP in your subnet, and insert under the ``SessionManagementSubscriptionData`` table:

   .. code-block:: sql

       INSERT INTO `SessionManagementSubscriptionData` (`ueid`, `servingPlmnid`, `singleNssai`, `dnnConfigurations`) VALUES
       ('<IMSI>', '00101', '{"sst": 1, "sd": "FFFFFF"}','{"oai":{"pduSessionTypes":{ "defaultSessionType": "IPV4"},"sscModes": {"defaultSscMode": "SSC_MODE_1"},"5gQosProfile": {"5qi": 6,"arp":{"priorityLevel": 15,"preemptCap": "NOT_PREEMPT","preemptVuln":"PREEMPTABLE"},"priorityLevel":1},"sessionAmbr":{"uplink":"10