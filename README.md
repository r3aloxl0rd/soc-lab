# SOC Lab Project
This project aims to simulate a Security Operations Center (SOC) environment using virtualization, where offensive tasks will be carried out and defensive actions will be demonstrated. It incorporates a SIEM solution with respective agents, vulnerable machines, and a proper network structure, seeking to emulate a realistic corporate environment.

# Structure
Below, I'll comment on the machines, software, vulnerabilities, and the network design, whilst underscoring some of my rationale.

## Network Design
The diagram below shows the network generally, including my chosen IP addresses.
```
┌─────────────────────────────────────────────────────────────┐
│                        HOST (Attacker)                      |
|                     192.168.100.1                           |
│                          CachyOS                            │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        │ dmz-net (192.168.100.0/24)
                        │
          ┌─────────────▼──────────────┐
          │        DVWA                │
          │   192.168.100.2            │
          │   Wazuh Agent              │
          └─────────────┬──────────────┘
                        │
                        │ internal-net (192.168.200.0/24)
                        │
          ┌─────────────▼──────────────┐
          │  ┌─────────────────────┐   │
          │  │     Internal 1      │   │
          │  │   192.168.200.2     │   │
          │  │   Wazuh Agent       │   │
          │  └─────────────────────┘   │
          │                            │
          │  ┌─────────────────────┐   │
          │  │       Wazuh         │   │
          │  │ 192.168.100.111     │   │
          │  │ 192.168.200.111     │   │
          │  └─────────────────────┘   │
          └────────────────────────────┘
```
### Internet / NAT
This part of the network is where outside attackers usually are (unless they're an insider threat, or are intentionally on the network as part of a white-box penetration test).
### DMZ Network
This part of the network is a demilitarized zone. It's where publicly-accessible servers usually sit.
### Internal Network
This part of the network is the internal corporate one. It's cut off from public access, and is only meant for internal services, such as those for employees.

## Machines
### Host
This is my own computer. It resembles the outside attacker, and only has direct visibility into DMZ machine(s).
### DVWA -- DMZ-Net Web-App Machine
Most front-facing services are web applications. I chose the Damn Vulnerable Web Application ([DVWA](https://github.com/digininja/DVWA)) because it's purposely vulnerable and thus can allow a seamless demonstration of different vulnerabilities and exploits, as well as what that would look like on a SIEM. This machine has a Wazuh agent installed, and has a NIC misconfiguration elaborated on further below.
### Internal 1 -- Internal-Net Employee 1
This is a poorly-configured Debian machine resembling that of an employee. It was not downloaded pre-built with vulnerabilities. This machine has a Wazuh agent installed.
### Wazuh -- DMZ & Internal-Net
This is an Ubuntu machine resembling the SOC server. It houses a SIEM solution known as Wazuh. It has direct visiblity into the internal and DMZ networks to allow monitoring.

## Choice of Software
### Operating Systems
I chose a mix of [Debian](https://www.debian.org/) and [Ubuntu](https://ubuntu.com/) in order to resemble a real environment where different distributions might be in use depending on need. I also chose CachyOS for my host to demonstrate that a normal Linux distribution can also conduct attacks, not necessarily penetration-testing-specific ones like Kali Linux or ParrotOS.
### SIEM
Wazuh was chosen because it's open-source, advanced, and easy to setup.
### Attacking Software
The only specialized piece of software I needed was [BurpSuite](https://portswigger.net/burp), in order to manipulate requests made to the DVWA machine. Otherwise, I did not need any specialized software to find out vulnerabilities or exploit them.

## Vulnerabilities
The vulnerabilities demonstrated in this lab project are well-known, common and easily reproducible. Not every potential attack vector is exploited, as that would exceed the scope of the project; the point is to showcase a straightforward attack path.
### File Uploads
File uploads are a common way to gain a foothold on server-side systems. This is demonstrated on 'DVWA'.
### sudo Misconfiguration
`sudo` misconfigurations can lead to many security problems, including allowing privilege escalation, lateral movement, installing unwanted packages, and much more. This is demonstrated on 'DVWA'.
### Misconfigured NIC
A misconfigured NIC can potentially allow dangerous lateral movement and pivoting. This is demonstrated on 'DVWA'.
### Anonymous FTP
File Transfer Protocol (FTP) is a largely deprecated protocol, and yet some may have it in use. In case anonymous login is enabled, it can allow accessing FTP-hosted files without credentials. This is demonstrated on 'Internal 1'.
### Weak Passwords
Weak passwords are the pillar of vulnerabilities. This is demonstrated on 'Internal 1'.
### Password Reuse
Reusing passwords across different accounts is another very dangerous vulnerabilities, typically leading to privilege escalation and lateral movement. This is demonstrated on 'Internal 1'.

## Instructions
If you wish to recreate this lab environment, or something similar to it, you can follow these general steps:
### Download Packages
At a minimum, you will need `libvirt`, `qemu`, `dnsmasq`, and `virt-manager`. Do not forget to enable/start any needed services, and enable hardware virtualization if it's not already enabled (usually from your UEFI/BIOS).
### Download Images
You'll need to acquire the images for the machines you wish to use. For example, in this lab, I needed a Debian ISO image and an Ubuntu ISO image.
### Set up QEMU/KVM and the Machines
I chose [`virt-manager`](https://virt-manager.org/) using QEMU instead of VirtualBox or VMWare for less overhead. The process is fairly straightforward for installing each VM itself (make sure you allocate adequate resources). Initially, unless configured otherwise, all VMs will be able to access the Internet. (If you're also going with DVWA for your vulnerable web-app, find instructions over at [its repository](https://github.com/digininja/DVWA))
### Defining Networks
You will need to construct an XML file defining the IP ranges for your networks. Next, define them using `virsh net-define <XML file>` and start them when needed (you can also set them to auto-start).
When on `virt-manager`, you add a NIC and select the associated network for each machine. However, initially, it is also convenient to add a NAT on alongside the isolated network interface, in order to allow updates and setup your Wazuh agents.
### Adding Wazuh 
Install Wazuh on the VM serving as your SIEM, and add agents using the VM's IP address (it should be on the same network as the machines you wish to monitor). The instructions are simple and displayed on Wazuh itself. Moreover, ensure that the log files you desire to read (such as `auth.log` and `access.log`) are both readable as well as able to be framed on Wazuh's dashboard; this can be a tedious process.
### Finalizing NICs
Once all desired Wazuh agents are deployed, updates are installed, and there is no longer a need for Internet access, you can cut off NAT interfaces from your non-public internal machines, by removing the hardware component on `virt-manager`.
### Bridging (Informational)
Your host will have visibility into all machines by default thanks to `libvirt`'s virtual bridges. You can delete some of these virtual bridges to enhance realism.
### Final Notes
Once all machines are properly set up, and logs are accurately reported, you can misconfigure things as you please (such as by modifying `/etc/sudoers`) and create your own attack demo. Logs should be collected on Wazuh demonstrating the SOC side.
