# SOC Lab Project
This project aims to simulate a Security Operations Center (SOC) environment using virtualization, where offensive tasks will be carried out and defensive actions will be demonstrated. It incorporates a SIEM solution with respective agents, vulnerable machines, and a proper network structure, seeking to emulate a realistic corporate environment.

# Structure
Below, I'll comment on the machines, software, vulnerabilities, and the network design, whilst underscoring some of my rationale.

## Network Design
The diagram below shows the network generally, including my chosen IP addresses.
```
┌─────────────────────────────────────────────────────────────┐
│                        HOST (Attacker)                      │
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
          │  │   192.168.200.3     │   │
          │  │   Wazuh Agent       │   │
          │  └─────────────────────┘   │
          │                            │
          │  ┌─────────────────────┐   │
          │  │     Internal 2      │   │
          │  │   192.168.200.4     │   │
          │  │   No Agent (Legacy) │   │
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
### DMZ-NET
This part of the network is a demilitarized zone. It's where publicly-accessible servers usually sit.
### Internal Network
This part of the network is the internal corporate one. It's cut off from public access, and is only meant for internal services, such as those for employees.

## Machines
### Host
This is my own computer. It resembles the outside attacker, and only has direct visibility into DMZ machine(s). I run CachyOS to demonstrate that any Linux distribution can conduct attacks, not necessarily penetration-testing-specific ones like Kali Linux or ParrotOS.
### DVWA -- DMZ-Net Front-Facing Machine
Most front-facing services are web applications. I chose the Damn Vulnerable Web Application (DVWA)(#https://github.com/digininja/DVWA) because it's very vulnerable and can allow a seamless demonstration of different vulnerabilities and exploits, as well as what that would look like on a SIEM. This machine has a Wazuh agent installed, and has a NIC misconfigurated elaborated on further below.
### Internal 1 -- Internal-Net Machine 1
This is a poorly-configured Debian machine resembling that of an employee. It was not downloaded pre-built with vulnerabilities. This machine has a Wazuh agent installed.
### Internal 2 -- Internal-Net Machine 2
This is a vulnerable Windows machine resembling that of an employee. This machine does not have a Wazuh agent installed due to compatibility issues, demonstrating that legacy hardware/software can get in the way of proper security.
### Wazuh -- DMZ & Internal-Net
This is an Ubuntu machine resembling the SOC server. It houses a SIEM solution known as Wazuh. It has direct visiblity into the internal and DMZ networks to allow monitoring.

## Choice of Software
### Operating Systems
I chose a mix of Debian, Ubuntu and Windows for the overall design in order to resemble a real environment where multiple OSs might be in use.
### SIEM
Wazuh was chosen because it's open-source, advanced, and easy to setup.
### Attacking Software
The only hacking-specific tool (or suite of tools) I needed was Metasploit. It is popular, powerful, and sufficient for this lab's needs.
## Vulnerabilities
The vulnerabilities demonstrated in this lab project are well-known, common and easily reproducible.
### EternalBlue
This is one of the most-damaging vulnerabilities Windows has encountered. This is demonstrated on 'Internal 2'.
### FTP
File Transfer Protocol (FTP) is a largely deprecated protocol, and yet some may have it in use. It has anonymous login enabled which means you can access FTP-accessible files without credentials. The FTP directory also has sensitive files stored on it. This is demonstrated on 'Internal 1'.
### Weak Passwords
Weak passwords are the pillar of vulnerabilities, and there's no shortage of them in this lab. This is demonstrated on 'Internal 1'.
### Connection from DMZ to Internal
I configured the DMZ machine to also have an interface allowing it to see internal machines. This would be a misconfiguration and allows lateral movement.

## Instructions
If you wish to recreate this lab environment, or something similar to it, you can follow these general steps:
### Download Packages
At a minimum, you will need `libvirt`, `qemu`, `dnsmasq`, and `virt-manager`. Do not forget to enable/start any needed services, and enable hardware virtualization if it's not already enabled (usually from your UEFI/BIOS).
### Download Images
You'll need to acquire the images for the machines you wish to use. For example, in this lab, I needed a Debian ISO file, an Ubuntu ISO file, and a Windows ISO file.
### Set up QEMU/KVM and the Machines
I chose virt-manager using QEMU instead of VirtualBox or VMWare for less overhead. The process is fairly straightforward for installing each VM itself (make sure you allocate adequate resources). Initially, unless configured otherwise, all VMs will be able to access the Internet.
### Defining Networks
You will need to construct an XML file defining the IP ranges for your networks. Next, define them using `virsh net-define <XML file>` and start them when needed (you can also set them to auto-start).
When on `virt-manager`, you add a NIC and select the associated network for each machine. However, initially, it is also convenient to add a NAT on alongside the isolated network interface, in order to allow updates and setup your Wazuh agents.
### Adding Wazuh 
Install Wazuh on the VM serving as your SIEM, and add agents using the VM's IP address (it should be on the same network as the machines you wish to monitor). The instructions are simple and displayed on Wazuh itself.
### Finalizing NICs
Once all desired Wazuh agents are deployed, updates are installed, and there is no longer a need for Internet access, you can cut off NAT interfaces from your non-public internal machines, by removing the hardware component on `virt-manager`.
### Bridging (Informational)
Your host will have visibility into all machines by default thanks to `libvirt`'s virtual bridges. You can delete some of these virtual bridges to enhance realism.
### Final Notes
Once all machines are properly set up, you can misconfigure them as you please and create your own attack demo. Logs should be collected on Wazuh demonstrating the SOC side.
