# Block 3 — DHCP and DNS

**Goal:** hand out addresses, and make sure clients can find the domain.

### 1. Install the role

```powershell
Install-WindowsFeature -Name DHCP -IncludeManagementTools
```

### 2. Authorize in AD

```powershell
Add-DhcpServerInDC -DnsName DC01.lab.local -IPAddress 192.168.x.x
Get-DhcpServerInDC
```

**A Windows DHCP server won't answer a single request until it's authorized.** It checks AD first.

That's real security — but it's **voluntary compliance**. It stops an unauthorized *Windows* DHCP server. It can't stop a home router, a Linux `dhcpd`, or a misconfigured appliance, because those never check. Blocking those needs **DHCP snooping** on the switch, which drops DHCP offers arriving on untrusted ports.

*AD authorization asks nicely. DHCP snooping enforces.*

### 3. Create the scope

```powershell
Add-DhcpServerv4Scope -Name "LAB-Scope" -StartRange 192.168.x.x -EndRange 192.168.x.x -SubnetMask 255.255.255.0 -State Active
Get-DhcpServerv4Scope
```

Starts at `.100`, leaving `.1–.99` for infrastructure that needs predictable addressing. `-State Active` because a scope can exist and be switched off.

### 4. Set the options

```powershell
Set-DhcpServerv4OptionValue -ScopeId 192.168.x.x -DnsServer 192.168.x.x -DnsDomain lab.local
Get-DhcpServerv4OptionValue -ScopeId 192.x.x
```

`-DnsServer` is **option 006** — every client gets told to resolve through DC01. Non-negotiable: a client pointed elsewhere can't find the domain.

`-DnsDomain` is **option 015**, the connection suffix — lets `dc01` resolve as shorthand for `dc01.lab.local`.

**Option 003 (gateway) deliberately unset.** Nothing to route to on an isolated network. Handing out a gateway that doesn't exist sends traffic nowhere.

`ScopeId` is the network address (`192.168.10.0`), which DHCP derived from the range and mask.

### 5. Reserve an address for the client

Get the MAC first — on the **host**:

```powershell
Get-VMNetworkAdapter -VMName WS01 | Select-Object MacAddress
```

Then on DC01:

```powershell
Add-DhcpServerv4Reservation -ScopeId 192.168.x.x -IPAddress 192.168.x.x -ClientId "00155D01D409" -Description "WS01"
Get-DhcpServerv4Reservation -ScopeId 192.168.x.x
```

Ties a MAC to a fixed IP while still leasing through DHCP. The address must fall **inside** the scope range — a reservation is a slice of the pool, not an exception to it.

Always set `-Description`. Twenty MACs with no labels is unusable in six months.

`00-15-5D` is Microsoft's OUI — every Hyper-V VM starts with it. VMware `00-50-56`, VirtualBox `08-00-27`.

<!-- ![DHCP scope, options and reservation](images/dhcp-scope.png) -->

### 6. Create DNS records by hand

**A record** — right-click zone → New Host (A). `fileserver` → `192.168.x.x`. Uncheck the PTR box; no reverse zone exists.

**CNAME** — right-click zone → New Alias. Alias name `file` (short), target `fileserver.lab.local` (**full FQDN**).

The asymmetry trips people up: the alias is created *inside* the zone so the suffix is implied, but the target could be anywhere, so it's spelled out.

```powershell
nslookup fileserver.lab.local
nslookup file.lab.local
```

The CNAME lookup shows the two-step: alias → canonical name → address.

Ignore `Server: UnKnown` and the reverse-lookup timeout — cosmetic, no reverse zone.

<!-- ![SRV records under _msdcs](images/dns-records.png) -->

### Record types

| Type | Answers |
|---|---|
| **A** | name → IPv4 address |
| **AAAA** | name → IPv6 address |
| **CNAME** | name → *another name* (two-step resolution) |
| **PTR** | address → name (needs a reverse zone) |
| **SRV** | *which server provides this service, on which port* |

**Why a CNAME.** `fileserver` is a machine's name; `files` is a job's name. Machines get replaced. Point everyone at the job name and a server swap is one record change, not reconfiguring every device.

**SRV is how a client finds a DC.** It doesn't know DC01 exists — it only knows it belongs to `lab.local`. So it asks:

```powershell
nslookup -type=SRV _ldap._tcp.dc._msdcs.lab.local
```

DNS answers with a name **and a port**. An A record can't carry a port; SRV can, plus priority and weight so several DCs share load and fail over.

**SRV doesn't authenticate anything.** It's a directory listing — find the server, then authenticate separately via Kerberos.

**Why DNS breaks logins.** Point a client at 8.8.8.8 and it asks "who provides LDAP for lab.local?" Google has never heard of it. The client never learns a DC exists. Network is fine, pings work, login fails.

*Most AD problems that look like network problems are DNS.*
