# Wazuh-Lab

# Introduction
The aim of this lab is to familiarise ourselves with Wazuh, The Hive and Shuffler in order to automate Security Operations Centre (SOC) tasks.
First, we will create a diagram to help visualise the data flow between our machines and components in the lab environment.

![Diagram](diagram.png)

Explanation:

1. The Windows 10 virtual machine (VM) sends Sysmon logs to the Wazuh server.
2. The Wazuh server sends this information to Shuffle if a certain rule is met, then runs a VirusTotal scan on the malware before creating a case in TheHive.
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

> Wazuh is a free, open-source security platform that unifies SIEM (Security Information and Event Management) and XDR (Extended Detection and Response) capabilities. It monitors endpoints, cloud workloads, and containers to detect threats, monitor file integrity, and ensure regulatory compliance. It uses a lightweight agent, a central server, and a dashboard for visualization.

You can use this [official documentation](https://documentation.wazuh.com/current/quickstart.html) to install and configure Wazuh, but for simplicity in this lab, we're going to use a [pre-built virtual machine image](https://documentation.wazuh.com/current/deployment-options/virtual-machine/virtual-machine.html) in Open Virtual Appliance (OVA) format which includes an Amazon Linux 2023 operating system and the Wazuh central components.

Let's navigate to VirtualBox > File > Import Appliance. Then we can select our pre-built, downloaded image, hit Next, and finish setting up our Wazuh server.

![wazuhova](/screenshots/wazuhova.png)

Next, we will set up and install TheHive.

# TheHive's Setup

> TheHive is a scalable, open-source, and commercial Security Incident Response Platform (SIRP) designed for SOCs, CERTs, and cybersecurity analysts to investigate, triage, and act upon security incidents collaboratively. It streamlines threat analysis by integrating with MISP (Malware Information Sharing Platform) and automating workflows, significantly reducing incident response time.

To setup TheHive, we're going to use a [Ubuntu 22.04.5 live server](https://releases.ubuntu.com/jammy/) virtual machine and follow [this documentation](https://docs.strangebee.com/thehive/installation/installation-guide-linux-standalone-server/) provided by StrangeBee.

# TheHive's Configuration

As the Hive's configuration is mentioned in the documentation, we won't go into much detail about it here. 
However, we are going to change some variables that were not mentioned in the documentation at the time of writing.

>  ⚠️ Note: Only run the following commands after installing Cassandra, Java and Elasticsearch from the documentation; otherwise, you won't be able to find the configuration files.

## Cassandra

Let's run:

```
nano /etc/cassandra/cassandra.yaml
```
Then change the parameters below:

![cluster](/screenshots/cluster.png)
![listenaddress](/screenshots/listenaddress.png)
![rpcaddress](/screenshots/rpcaddress.png)
![seeds](/screenshots/seeds.png)

> Click ```Ctrl+W``` if you want to search for a specific parameter or word.

> To find out your IP address, run the ```ip a``` command then make the necessary changes to the ```listen_address``` , ```rpc_address``` and ```seeds``` fields.

Run: 

```
systemctl stop cassandra.service
rm -rf /var/lib/cassandra/*
systemctl start cassandra.service
```

## ElasticSearch

```
nano /etc/elasticsearch/elasticsearch.yml
```
Change these parameters:

![ecluster](/screenshots/eclustername.png)
![enode](/screenshots/enode.png)
![eip](/screenshots/eip.png)
![emaster](/screenshots/emaster.png)

Run:
```
systemctl start elasticsearch
systemctl enable elasticsearch
```

## TheHive
Run: 
```
chown -R thehive:thehive /opt/thp
nano /etc/thehive/application.conf
```
Change:

![thehiveip](/screenshots/thehiveip.png)
![thehivebaseurl](/screenshots/thehivebaseurl.png)

Run:
```
systemctl start thehive
systemctl enable thehive
```



Once we have configured Cassandra, Elasticsearch and TheHive, let's check if our services are active by running the following commands:
```
systemctl status thehive
systemctl status cassandra.service
systemctl status elasticsearch.service
```

If all services are up and active, we can access our server by entering its IP address and port 9000 into our browser's URL.

![thehivelogin](/screenshots/thehivelogin.png)

# Windows 10 Wazuh Agent

To get logs from our Windows 10 machine, we must first set up a Wazuh agent on it. 

![wazuhagent](/screenshots/wazuhagent1.png)
![wazuhagent](/screenshots/wazuhagent2.png)
![wazuhagent](/screenshots/wazuhagent3.png)

Let's copy the PowerShell command we received from Wazuh to our Windows 10 machine:

```
Invoke-WebRequest -Uri https://packages.wazuh.com/4.x/windows/wazuh-agent-4.14.3-1.msi -OutFile $env:tmp\wazuh-agent; msiexec.exe /i $env:tmp\wazuh-agent /q WAZUH_MANAGER='192.168.1.32' WAZUH_AGENT_NAME='demo'
```

Then start the Agent Service:
```
NET START Wazuh
```

To feed Sysmon logs to the Wazuh server on our Windows machine, we need to make some changes to the Wazuh agent configuration file ```ossec.conf```, which is located in ```C:\Program Files (x86)\ossec-agent```

Scroll down through the file and under ```<!-- Log analysis -->```, lets add this :

```
<localfile>
    <location>Microsoft-Windows-Sysmon/Operational</location>
    <log_format>eventchannel</log_format>
  </localfile>
```
![ossec](/screenshots/ossec.png)

Restart the Wazuh service:
```
net stop wazuh
net start wazuh
```
Our Sysmon logs can now appear in our Wazuh's dashboard.

# Telemetry
If we were to create telemetry using the current configurations, Wazuh might not display the telemetry because it does not log all events by default. Fortunately, we can change this by accessing the file ```/var/ossec/etc/ossec.conf``` on our Wazuh hosting machine and making some changes.

![ossecconf](/screenshots/ossecconf.png)

Then restart the service: 
```
systemctl restart wazuh-manager.service
```

This forces Wazuh to archive all logs and store them in a single file called ```archives```, located in the ```/var/ossec/logs/archives``` directory.

In order for us to start ingesting these logs, we need to change the configuration file in ```/filebeat```. To do that, let's run:
```
nano /etc/filebeat/filebeat.yml
```
![filebeat](/screenshots/filebeat.png)

Then restart the service:
```
systemctl restart filebeat
```
Now let’s go back to the Wazuh dashboard and create a new index pattern to read the wazuh-archives-* logs.

![index1](/screenshots/index1.png)
![index2](/screenshots/index2.png)

# Real Telemetry

Now that Wazuh can read all the events, let's make some noise by running [Mimikatz](https://github.com/gentilkiwi/mimikatz/releases/tag/2.2.0-20220919) on our Windows 10 virtual machine.
> Mimikatz is a widely used, open-source security tool created by Benjamin Delpy that extracts plain-text passwords, hashes, PINs, and Kerberos tickets from Microsoft Windows memory. It is primarily used to analyze Windows security, but also to perform attacks like pass-the-hash or privilege escalation.

Let's open a PowerShell window, navigate to the directory where Mimikatz is installed, and then run the ```mimikatz.exe```.

![mimikatz](/screenshots/mimikatz.png)

This should generate some logs in the Wazuh dashboard. Let's go back and check.
> Note: Use the ```wazuh-archives-**``` index pattern we created earlier.

![mimikatzlog](/screenshots/mimikatzlog.png)

Bingo!

# Creating Custom Rules
In Wazuh, a rule is a condition that is used to detect important or suspicious events in log files and generate an alert.
To create an alert that prompts a Security Operations Centre (SOC) analyst to take action, we will create a custom rule that detects the start of our Mimikatz process.
Head to Server Management > Rules > Manage Rules > Custom Rules > ```local_rules.xml```.
You can now obtain a Wazuh Rule for Sysmon Event ID 1 by using ChatGPT or copying a rule from ```0800-sysmon_id_1.xml``` to customise your own rule. In this lab, we will use the following rule from ```0800-sysmon_id_1.xml```:
```
<rule id="92000" level="4">
    <if_group>sysmon_event1</if_group>
    <field name="win.eventdata.parentImage" type="pcre2">(?i)\\(c|w)script\.exe</field>
    <options>no_full_log</options>
    <description>Scripting interpreter spawned a new process</description>
    <mitre>
      <id>T1059.005</id>
    </mitre>
  </rule>
```
Let's tailor it to our preferences — we should end up with something like this:

![localrule](/screenshots/localrule.png)









