# Active Directory Home Lab

A fully functional enterprise Active Directory environment built from scratch on Microsoft Azure, demonstrating real-world network administration skills.

##  Infrastructure

| Component | Details |
|---|---|
| **Cloud Platform** | Microsoft Azure |
| **Domain Controller** | Windows Server 2022 (Standard_D2s_v3) |
| **Client Machine** | Windows 11 Pro (Standard_D2s_v3) |
| **Domain** | corp.local |

##  OU Structure

```
corp.local
└── Corp
    ├── Users
    │   ├── HR
    │   ├── IT
    │   ├── Sales
    │   └── Finance
    ├── Groups
    └── Computers
        ├── Workstations
        └── Servers
```

##  Users & Groups

- **11 domain users** created across 4 departments via PowerShell bulk import (CSV)
- **AGDLP group model** implemented for role-based access control:
  - Global groups by department: `HR_Users`, `IT_Users`, `Sales_Users`, `Finance_Users`, `IT_Admins`
  - Domain Local groups by resource: `FileShare_HR_FullAccess`, `FileShare_Finance_FullAccess`, etc.

### What is AGDLP?
Accounts → Global groups → Domain Local groups → Permissions. Microsoft's best-practice model for scalable access control. Users are added to Global groups by role. Global groups are nested into Domain Local groups that hold resource permissions. When someone joins or leaves, only the Global group membership changes — resource permissions stay untouched.

##  Group Policy Objects

| GPO | Linked To | What It Does |
|---|---|---|
| `Corp Password Policy` | Domain root | Min 12 chars, complexity, 90-day expiry, 5-attempt lockout |
| `Auto Lock Screen` | Corp → Users OU | Screen saver timeout at 5 min, password protected |
| `Finance - Block Control Panel` | Finance OU | Restricts Control Panel access for Finance users only |
| `Deploy 7-Zip` | Workstations OU | Pushes 7-Zip MSI to all workstations via software deployment |
| `AD Auditing Policy` | Domain Controllers OU | Enables auditing of logon events, account management, directory changes |

##  Software Deployment via GPO

Deployed 7-Zip enterprise-wide using Group Policy software installation:
- Hosted `.msi` installer on a UNC share (`\\DC01\Software`)
- Created Computer Configuration GPO with Assigned package
- Linked to Workstations OU
- Verified automatic installation on CLIENT01 at next boot (no manual install)

##  AD Auditing

Configured auditing policy capturing:

| Event ID | Description |
|---|---|
| 4624 | Successful logon |
| 4625 | Failed logon attempt |
| 4740 | Account lockout |
| 4738 | User account modified |
| 4728 | User added to security group |


##  Screenshots

- `ou-structure.png` — ADUC showing full OU tree with users
- `gpo-list.png` — GPMC showing all linked GPOs
- `audit-log.png` — Event Viewer filtered Security log 
- `7zip-deployed.png` — 7-Zip installed on CLIENT01 via GPO

##  Scripts

### Bulk User Creation (`scripts/bulk-create-users.ps1`)
```powershell
$users = Import-Csv "C:\Users\azureadmin\Desktop\new_users.csv"
$defaultPassword = ConvertTo-SecureString "Welcome2026!" -AsPlainText -Force

foreach ($user in $users) {
    $fullName = "$($user.FirstName) $($user.LastName)"
    $samName = "$($user.FirstName.ToLower()).$($user.LastName.ToLower())"
    $upn = "$samName@corp.local"
    $ouPath = "OU=$($user.OU),OU=Users,OU=Corp,DC=corp,DC=local"

    New-ADUser `
        -Name $fullName `
        -GivenName $user.FirstName `
        -Surname $user.LastName `
        -SamAccountName $samName `
        -UserPrincipalName $upn `
        -Path $ouPath `
        -AccountPassword $defaultPassword `
        -Department $user.Department `
        -Enabled $true `
        -PasswordNeverExpires $true

    Write-Host "Created user: $fullName in $($user.OU) OU" -ForegroundColor Green
}
```

##  Network Diagram

```
                    [Internet]
                        |
              [Azure Virtual Network]
              (10.0.0.0/24 - West US )
                        |
          +-------------+-------------+
          |                           |
    [DC01 - 10.0.0.4]         [CLIENT01 - 10.0.0.5]
    Windows Server 2022         Windows 11 Pro
    AD DS, DNS, DHCP            Domain joined
    corp.local                  CORP\sarah.johnson
```
