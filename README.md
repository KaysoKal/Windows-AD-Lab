# Windows Server 2022 Active Directory Home Lab

A working Active Directory domain built from scratch in Hyper-V — domain controller, DNS, DHCP, Group Policy, PowerShell automation, delegated administration, and BitLocker with recovery keys escrowed to AD.

## About this lab

I kept seeing the same requirements in job postings — Active Directory, Group Policy, PowerShell, Intune, imaging — and I had the certifications but not the hands-on time. This lab was built to close that gap deliberately: pick the skills employers actually list, build an environment that exercises each one, and be able to walk through any of it in an interview.

## Environment

| Role - Name | OS | Address |
|---|---|---|---|
| Hyper-V host | on my own computer | Windows 11 Pro, 32 GB RAM | — |
| Domain controller | DC01 | Windows Server 2022 Standard (Desktop Experience) | 192.168.10.10 static |
| Client workstation | WS01 | Windows 11 Enterprise | 192.168.10.150 DHCP reservation |

## OU structure 

Lab.local 
├── LAB-Users
│   ├── Warehouse
│   ├── Office
│   └── IT
├── LAB-Computers
│   ├── Warehouse-Workstations
│   ├── Office-Workstations
│   └── Handhelds
├── LAB-Groups
├── LAB-ServiceAccounts
└── LAB-Disabled
Users and computers live in a separate ou on purpose.  

**Active Directory**
New forest promotion, the OU structure above, 19 user accounts with Department and Title populated, and four Global security groups with membership assigned by department.

**DNS and DHCP**
DNS installed during promotion, with forwarders added so the domain controller can resolve external names. DHCP authorized in AD with a scope of 192.168.10.100–200, options 006 and 015 set, and a reservation pinning WS01 to .150 by MAC address. A and CNAME records created by hand to work with each record type.

**Group Policy** — five GPOs:

| GPO | Linked to | Effect |
|---|---|---|
| Password Policy | Domain root | 14-character minimum, complexity on, no expiry, 5-attempt lockout |
| Screen Lock | LAB-Computers | Locks after 10 minutes idle |
| USB Block | Warehouse-Workstations | Denies all removable storage |
| Warehouse Drive Map | Warehouse users | Maps H: to a shared folder |
| Printer Deploy | Office users | Deploys a shared printer |

**Delegated administration**
Password-reset rights granted to the `SEC-IT-Helpdesk` group, scoped to the Warehouse OU only. Verified two ways — with `Get-Acl` on the OU, and by logging in as a delegated user and confirming the same command succeeded in Warehouse and was denied in IT.

**BitLocker encrypted with no recovery key.**
Encryption completed successfully — and `Get-BitLockerVolume` showed only a TPM protector, no recovery password. Nothing to escrow, and no way to recover the drive if the TPM ever failed. Creating a key and storing a key are two separate operations.

## Skills demonstrated

Windows Server 2022 · Active Directory Domain Services · DNS · DHCP · Group Policy · PowerShell scripting · delegated administration and least privilege · BitLocker and key escrow · Hyper-V virtualization · Microsoft Entra ID · Intune

