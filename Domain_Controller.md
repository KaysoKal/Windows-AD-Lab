# Block 1 — Domain Controller

**Goal:** turn a plain Windows Server into the machine that decides who exists and who's allowed in.

### 1. Find the network adapter

```powershell
Get-NetAdapter
```

Need its exact name for the next two commands. Usually `Ethernet`.

### 2. Set a static IP

```powershell
New-NetIPAddress -InterfaceAlias "Ethernet" -IPAddress 192.168.10.10 -PrefixLength 24 -AddressFamily IPv4
```

A DC can't use DHCP — clients need it at a known address. `-PrefixLength 24` is the /24. **No gateway** — the lab network is isolated, nothing to route to.

### 3. Point DNS at itself

```powershell
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses 127.0.0.1
```

`127.0.0.1` is loopback — this machine. A DC must resolve through its own DNS, because that's where the SRV records live that clients use to find it.

### 4. Verify

```powershell
Get-NetIPConfiguration
```

Want: `.10`, no gateway, DNS `127.0.0.1`.

<!-- ![Static IP configured](images/static-ip.png) -->

### 5. Rename

```powershell
Rename-Computer -NewName DC01 -Restart
```

**Before promotion.** After promotion the name is baked into AD objects, DNS records and Kerberos SPNs. Renaming a live DC is painful.

Confirm with `$env:COMPUTERNAME`.

### 6. Install the AD DS role

```powershell
Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools
```

Puts the software on the machine. It *can* be a DC now — it isn't one yet.

`-IncludeManagementTools` gets ADUC and the AD PowerShell module. Forget it and you have the role with no way to manage it.

### 7. Promote — build the domain

```powershell
Install-ADDSForest -DomainName "lab.local" -DomainNetbiosName "LAB" -InstallDns
```

This is what actually creates the domain. It builds:

- `ntds.dit` — the database holding every user, computer and group
- the DNS zone, including the `_msdcs` SRV records clients query to find a DC
- SYSVOL — the share holding Group Policy and logon scripts

The server stops being standalone. Prompts for a **DSRM password** (repair-mode account — write it down, it's unrecoverable). Reboots itself.

**Role ≠ promotion.** Step 6 installs software; step 7 creates the domain. The same role serves three outcomes — new forest, new domain in an existing forest, or additional DC — which is why the choice happens here.

### 8. Log in

`LAB\Administrator` — a domain account now, not a local one.

### 9. Verify promotion

```powershell
nltest /dsgetdc:lab.local
```

Asks the same question a client asks at login: *who is the DC for lab.local?* A clean answer proves AD is running, DNS is serving, and the SRV records point right.

Flags to recognise: **PDC** (also the domain's time source) · **GC** (Global Catalog) · **KDC** (Kerberos tickets) · **TIMESERV**.

```powershell
dcdiag /v | Select-String "failed"
```

Silence = all tests passed. `dcdiag` reads the event log, so it reports history — after a reboot it may show stale startup-race errors. Check live state with `Get-Service DHCPServer,DNS,NTDS`.

<!-- ![nltest showing DC flags](images/dc-promoted.png) -->

### 10. Checkpoint

```powershell
Checkpoint-VM -Name DC01 -SnapshotName "Promoted-Clean"
```

Run on the **host**, not the VM.

Snapshot = save point. Rollback = returning to it. Good for short-term undo before a risky change. **Not a backup** — same host, not meant to persist.

**For a DC: restore from backup, don't roll back.** Rewinding reuses sequence numbers other DCs have seen, they assume they're caught up, replication silently stops. Safe here with one DC.

---
