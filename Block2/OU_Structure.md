#Verify

---
**Telling them apart:**

```powershell
# Your OUs — the built-in containers won't appear here
Get-ADOrganizationalUnit -Filter * | Select-Object Name,DistinguishedName

# The built-in containers
Get-ADObject -Filter 'ObjectClass -eq "container"' -SearchScope OneLevel -SearchBase "DC=lab,DC=local" |
    Select-Object Name,DistinguishedName
```

The quick tell is the prefix: containers start with `CN=` (`CN=Users,DC=lab,DC=local`), OUs start with `OU=`.

**Finding objects stuck in a default container:**

```powershell
Get-ADComputer -Filter * -SearchBase "CN=Computers,DC=lab,DC=local" | Select-Object Name
Get-ADUser -Filter * -SearchBase "CN=Users,DC=lab,DC=local" | Select-Object Name
```

Anything the first command returns is a machine no GPO can reach — the first check when "new PCs aren't getting policy."
