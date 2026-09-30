#Verify

```powershell
Get-ADOrganizationalUnit -Filter * | Select-Object Name,DistinguishedName
```

<!-- ![OU tree in ADUC](images/ou-structure.png) -->

### What matters here

**An OU is a container you can point policy at.** The built-in `Users` and `Computers` containers exist by default but **can't have GPOs linked to them** — that's the entire reason for building your own.

**Users and computers separated on purpose.** Every GPO has a Computer half and a User half. Separate OUs means each GPO targets one object type cleanly.

**`-Filter` is required on AD cmdlets.** Leaving it off errors rather than returning everything.

**DistinguishedName reads right to left:**

```
OU=Warehouse,OU=LAB-Users,DC=lab,DC=local
```

`CN` = leaf object (user/group/computer) · `OU` = container · `DC` = domain component.

Scripts build these strings from data, so the format matters.

---
