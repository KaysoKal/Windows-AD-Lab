**Goal:** populate the directory, and control access by role rather than by person.

### 1. Create users in ADUC

Ten users by hand first — so I'd know what fields the wizard writes before automating it on Saturday.

Right-click the target OU → New → User. Set **Department** and **Title** under Properties afterward.

Logon name pattern: `firstname.lastname`. Consistent, predictable, and what the Saturday script generates.

### 2. Create the security groups

In `LAB-Groups`: right-click → New → Group. **Security**, **Global** scope.

`SEC-Warehouse-Staff` · `SEC-Office-Staff` · `SEC-IT-Helpdesk` · `SEC-Drive-Warehouse`

### 3. Add members

```powershell
Add-ADGroupMember -Identity "SEC-Warehouse-Staff" -Members john.walker,maria.santos,james.coleman,tanya.brooks
```

`-Members` takes the **logon names** (SamAccountName), comma-separated — not display names.

### 4. Verify

```powershell
Get-ADUser -Filter * -SearchBase "OU=LAB-Users,DC=lab,DC=local" -Properties Department,Title |
    Select-Object Name,Department,Title | Sort-Object Department

Get-ADGroup -Filter * -SearchBase "OU=LAB-Groups,DC=lab,DC=local" |
    Select-Object Name,GroupScope,GroupCategory

Get-ADGroupMember -Identity "SEC-Warehouse-Staff" | Select-Object Name
```

<!-- ![Users sorted by department](images/users-by-department.png) -->

### What matters here

**`-Properties` is required for anything outside the default set.** Department, Title, LockedOut and LastLogonDate come back blank without it — which looks like missing data rather than an unrequested field.

**Security groups grant permissions. Distribution groups are email only** and can't control access to anything.

**Group scope.** *Global* holds users from its own domain and works anywhere in the forest — natural for role-based groups of people. *Domain Local* can hold anyone but only grants access in its own domain — natural for attaching to a resource. The pattern is **AGDLP**: Accounts into Global groups, Global groups into Domain Local groups, Permissions on the Domain Local.

**Grant to the group, not the person.** New hire joins the group and inherits everything; leaver drops out and loses it all at once.

---

## Verify the whole day

```powershell
dcdiag /v | Select-String "failed"
nltest /dsgetdc:lab.local
Get-ADOrganizationalUnit -Filter * | Select-Object Name
Get-DhcpServerInDC
Get-DhcpServerv4Scope
Get-DhcpServerv4OptionValue -ScopeId 192.168.x.x
Get-DhcpServerv4Reservation -ScopeId 192.168.x.x
Get-ADUser -Filter * -SearchBase "OU=LAB-Users,DC=lab,DC=local" | Measure-Object
```
