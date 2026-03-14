# Wazuh-Lab

# Introduction
The aim of this lab is to familiarise ourselves with Wazuh, The Hive and Shuffler in order to automate Security Operations Centre (SOC) tasks.
First, we will create a diagram to help visualise the data flow between our machines and components in the lab environment.

![Diagram](diagram.png)

Explanation:

1. The Windows 10 virtual machine (VM) sends Sysmon logs to the Wazuh server.
2. The Wazuh server sends this information to Shuffle, which runs a VirusTotal scan on the malware before creating a case in TheHive.
3. An email alert is sent to the analyst to notify them of the event, allowing them to investigate and take action.
