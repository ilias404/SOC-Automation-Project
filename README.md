# Wazuh-Lab

# Introduction
The aim of this lab is to familiarise ourselves with Wazuh, The Hive and Shuffler in order to automate Security Operations Centre (SOC) tasks.
First, we will create a diagram to help visualise the data flow between our machines and components in the lab environment.

![Diagram](diagram.png)

Explanation:

1. The Windows 10 virtual machine (VM) sends Sysmon logs to the Wazuh server.
2. The Wazuh server sends this information to Shuffle, which runs a VirusTotal scan on the malware before creating a case in TheHive.
3. An email alert is sent to the analyst to notify them of the event, allowing them to investigate and take action.

# Setup

Let's download and configure sysmon on our Windows 10 virtual machine.

> **Sysmon (System Monitor)** is a Windows system service and device driver that logs system activity to the Windows Event Log. It's part of the Microsoft Sysinternals Suite and is widely used for advanced event logging.

We’ll install it and configure it using **Olaf Hartong’s `sysmonconfig.xml`**, a community-trusted configuration.

### Step 1: Download Sysmon

1. Visit the official [Microsoft Sysinternals Sysmon page](https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon)
2. Download **Sysmon for Windows**
3. Extract the zip contents

### Step 2: Download Olaf’s Sysmon Config

1. Visit Olaf’s GitHub repo: [https://github.com/olafhartong/sysmon-modular](https://github.com/olafhartong/sysmon-modular)
2. Download the `sysmonconfig.xml` file to the directory where Sysmon is downloaded.

### Step 3: Install and Configure Sysmon

Open a **Powershell as Administrator** and navigate to the folder where `Sysmon64.exe` and `sysmonconfig.xml` are located.

Then run:

```cmd
.\Sysmon64.exe -i sysmonconfig.xml
```
Once installed, our Sysmon service should be ready to use. We can now ingest the Sysmon logs into our Wazuh server.

# Wazuh Server Setup

You can use this [official documentation](https://documentation.wazuh.com/current/quickstart.html) to install and configure Wazuh, but for simplicity in this lab, we're going to use a [pre-built virtual machine image](https://documentation.wazuh.com/current/deployment-options/virtual-machine/virtual-machine.html) in Open Virtual Appliance (OVA) format which includes an Amazon Linux 2023 operating system and the Wazuh central components.


