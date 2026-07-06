# 🛡️ My Enterprise Active Directory & Network Hardening Homelab

![Windows Server](https://img.shields.io/badge/Windows_Server-0078D6?style=for-the-badge&logo=windows&logoColor=white)
![Active Directory](https://img.shields.io/badge/Active_Directory-0078D6?style=for-the-badge&logo=windows&logoColor=white)
![PowerShell](https://img.shields.io/badge/PowerShell-5391FE?style=for-the-badge&logo=powershell&logoColor=white)
![VirtualBox](https://img.shields.io/badge/VirtualBox-183A61?style=for-the-badge&logo=virtualbox&logoColor=white)

## Welcome to my project.
I built this Active Directory homelab from the ground up to get hands-on experience with enterprise infrastructure and identity access management (IAM). My main goal wasn't just to set up a network, but to actively defend it. 

After provisioning the environment in an isolated VirtualBox network, I implemented a strict security baseline using Group Policy Objects (GPOs) to mitigate common cyber threats like brute-force attacks, lateral movement, and credential dumping.

## The Lab Architecture
To simulate a real corporate environment safely (and without blowing up my own home network), I built everything on an internal NAT Network (`10.0.2.0/24`) using VirtualBox.

* **The Brains (DC-01):** Windows Server 2022 acting as the Domain Controller (Static IP: `10.0.2.10`).
* **The Endpoint (WKSTN-01):** Windows 10 Enterprise, joined to the domain.
* **The Domain:** `corp.local`

## How I Hardened the Network
Once the core infrastructure was talking, I went into Group Policy Management and started locking things down. Here are the specific security controls I deployed across the domain:

* **Password Complexity & Lockouts:** Configured a GPO requiring 12+ character passwords and enforced a 15-minute account lockout after 5 failed attempts. *Goodbye, basic brute-force attacks.*
* **Centralized Firewall Enforcement:** Standard users love to turn off their firewalls. I used a GPO to force Windows Defender Firewall to stay "ON" for Domain, Private, and Public profiles globally.
* **Killed Legacy Protocols:** Disabled LLMNR and NetBIOS node types. These are notorious for being abused in network poisoning and Man-in-the-Middle (MitM) credential relay attacks (like Responder).
* **Principle of Least Privilege:** Ensured standard user accounts have absolutely zero local administrative rights on their workstations to limit the blast radius of a potential compromise.
* **LAPS Deployment:** Deployed Microsoft Local Administrator Password Solution (LAPS) to randomize the local admin passwords on endpoints, effectively neutralizing Pass-the-Hash (PtH) and lateral movement tactics.

## Action Shots & Proof of Concept

### 1. Forcing the Firewall via GPO
Here is the Group Policy configuration ensuring endpoints cannot be exposed to unauthorized traffic by a rogue user or standard malware.

https://keep.google.com/u/0/media/v2/15If82ofNMhNZoxfjDPqo1OGjWv-6xp1uJ42jUYjiE3C2MHYcDChBHRsNH0lUEA/1mYFUSiex8eQq7RI_PQe1JbCOpDs23FIDyBdGZ5ehsPYY6LGLU2JogXOUiysylKI?sz=512&accept=image%2Fgif%2Cimage%2Fjpeg%2Cimage%2Fjpg%2Cimage%2Fpng%2Cimage%2Fwebp
![Windows Defender Firewall GPO Configuration](./images/firewall-gpo.png)

### 2. Validating the Lockout Policy
Security policies don't matter if they don't actually work. I intentionally spammed bad passwords against a dummy user account. As you can see in the Event Viewer, the Active Directory domain successfully caught it and logged **Event ID 4740 (A user account was locked out)**.


![Event Viewer - Event ID 4740 Account Lockout](./images/event-id-4740.png)

## 🧠 Lessons Learned & Takeaways
This project was incredibly fun, but homelabbing always comes with a "troubleshooting tax." Getting the DNS settings perfectly aligned so the workstation could actually find and join the `corp.local` domain was a great learning experience in networking fundamentals. 

Ultimately, this lab reinforced the concept of **Defense in Depth**. It showed me firsthand how native Windows tools, when configured correctly, can drastically shrink an organization's attack surface before you even need to buy expensive third-party security software.
