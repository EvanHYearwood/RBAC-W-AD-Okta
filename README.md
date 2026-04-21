# AD + Okta RBAC Lab
**Yearwood.Local | Active Directory + Okta Identity Management**

Hi,
In this hands-on IAM lab I implement Role-Based Access Controls (RBAC) by integrating an on-premises Active Directory domain with Okta as the identity provider.

<h2>Video Demonstration</h2>

- ### [Watch me do a Live Lab Debrief 🚀](https://youtu.be/V3FpYYJUDQU)
---

## Table of Contents
1. [Lab Overview](#lab-overview)
2. [Environment](#environment)
3. [RBAC Architecture](#rbac-architecture)
4. [Workflow: Step-by-Step](#workflow-step-by-step)
   - [Phase 1 — Active Directory Setup](#phase-1--active-directory-setup)
   - [Phase 2 — Okta Integration](#phase-2--okta-integration)
   - [Phase 3 — Access Sprawl Investigation](#phase-3--access-sprawl-investigation)
   - [Phase 4 — Remediation & Rebuild](#phase-4--remediation--rebuild)
   - [Phase 5 — RBAC Implementation & Verification](#phase-5--rbac-implementation--verification)
5. [Key Lessons Learned](#key-lessons-learned)
6. [Tools & Technologies](#tools--technologies)

---

## Lab Overview

The goal of this lab was to implement a clean RBAC model where:
- Active Directory is the **source of truth for identity** (who you are)
- Okta is the **access policy layer** (what you can use)
- Users are assigned to **role groups in AD** — never directly to apps
- App access is granted **exclusively through group membership**

The lab encountered real-world access sprawl issues caused by misconfiguration and resolved them through API-based remediation and architectural rebuilding.

<img width="687" height="754" alt="RBAC Lab | March 2026" src="https://github.com/user-attachments/assets/2f743507-8c4a-456b-9e8b-760d2e3c1b6d" />


---

## Environment

| Component | Detail |
|-----------|--------|
| Domain | yearwood.local |
| Domain Controller | Windows Server (on-prem) |
| Okta Tenant | Okta trial org |
| Admin Account | Eva Admin |
| Users | 14 domain users across 4 roles |
| Applications | Gworkspace, SharePoint, Slack (SAML) |

---

## RBAC Architecture

### Role Groups (Active Directory — Source of Truth)

| Group | Members |
|-------|---------|
| V2_Finance | 4 users — Finance department |
| V2_Helpdesk | 4 users — Helpdesk department |
| V2_HR | 4 users — HR department |
| V2_Cloud_Engineer | 4 users — Engineering team |

### Access Matrix

| Role | Gworkspace | SharePoint | Slack |
|------|-----------|------------|-------|
| V2_Finance | ❌ | ✅ | ✅ |
| V2_Helpdesk | ❌ | ✅ | ✅ |
| V2_HR | ✅ | ✅ | ✅ |
| V2_Cloud_Engineer | ✅ | ✅ | ✅ |

### Clean RBAC Pipeline
```
AD User → AD Role Group → Okta Group Sync → Group-to-App Assignment → Application
```
One pipeline. No direct user-to-app assignments. No nested groups.

---

## Workflow: Step-by-Step

### Phase 1 — Active Directory Setup

**Step 1: Build the OU Structure**

Created the following Organizational Unit structure in Active Directory Users and Computers (ADUC):

```
yearwood.local
└── _NA
    ├── Groups
    ├── Users
    ├── Admins
    ├── DisabledUsers
    ├── ServiceAccounts
    └── Workstations
```

> 📸 *[Screenshot: ADUC showing OU structure]*
> <img width="760" height="535" alt="image" src="https://github.com/user-attachments/assets/b2399fea-0023-4251-be36-18a0c6183b4f" />


---

**Step 2: Create Role Groups in AD**

Created 4 Security Groups under `_NA > Groups`:

```powershell
New-ADGroup -Name "V2_Finance"        -GroupScope Global -GroupCategory Security -Path "OU=Groups,OU=_NA,DC=yearwood,DC=local"
New-ADGroup -Name "V2_Helpdesk"       -GroupScope Global -GroupCategory Security -Path "OU=Groups,OU=_NA,DC=yearwood,DC=local"
New-ADGroup -Name "V2_HR"             -GroupScope Global -GroupCategory Security -Path "OU=Groups,OU=_NA,DC=yearwood,DC=local"
New-ADGroup -Name "V2_Cloud_Engineer" -GroupScope Global -GroupCategory Security -Path "OU=Groups,OU=_NA,DC=yearwood,DC=local"
```

> ⚠️ **Lesson Learned:** Do NOT create application groups (GG_Slack, GG_VPN, etc.) in AD. Application entitlement groups should belong in Okta. Creating them in AD and nesting them inside role groups caused three sessions of access sprawl.

---

**Step 3: Create Users and Assign to Role Groups**

Created 14 domain users under `_NA > Users` and bulk-assigned each to exactly one role group via PowerShell.

```powershell
# Example: Assign users to a role group
Add-ADGroupMember -Identity "V2_Finance" -Members "user1","user2","user3","user4"
```

> 📸 *[Screenshot: ADUC showing users in Groups OU]*
> <img width="844" height="556" alt="image" src="https://github.com/user-attachments/assets/989ed1b2-5b19-4819-824e-1120a4bd7af4" />


---

### Phase 2 — Okta Integration

**Step 4: Connect Active Directory to Okta**

1. In Okta Admin Console: **Directory > Directory Integrations > Add Active Directory**
2. Downloaded and installed the Okta AD Agent on the domain controller
3. Configured the agent to point to `yearwood.local`
4. Selected the `_NA` OU for import scope

---

**Step 5: Run Full Import**

1. **Directory > Directory Integrations > yearwood.local > Import Now > Full Import**
2. Waited for import to complete
3. Reviewed pending items in the **Import tab**
4. Clicked **Confirm Assignments**

> ⚠️ **Critical Rule:** Only run Full Import when AD is in a clean, final state. Never run Incremental Import during initial setup or while making structural changes — Incremental only adds, never removes, and will lock in bad memberships if confirmed.

---

**Step 6: Add SAML Applications**

Added three SAML app integrations from the Okta Integration Network:
- Google Workspace
- Microsoft SharePoint
- Slack

Configured each with the appropriate SAML settings for yearwood.local.

> 📸 *[Screenshot: Okta Applications list showing 3 SAML apps]*
> <img width="1255" height="786" alt="image" src="https://github.com/user-attachments/assets/7d7d04c2-d60e-400e-ad22-e19cd0ef9dc4" />


---

### Phase 3 — Access Sprawl Investigation

> This phase documents the problems encountered and how they were diagnosed. Understanding what went wrong is as valuable as the solution.

**What Went Wrong**

Early in the lab, application groups were created in AD and nested inside role groups. This created three independent access paths in Okta simultaneously:

```
Path 1: AD nesting → Okta inherited memberships (Okta-managed, not AD-managed)
Path 2: Direct user-to-app assignments by Eva Admin during SAML testing (163 events)
Path 3: Orphaned Okta groups with no AD directory linkage
```

Each path granted access independently. Fixing one left the other two intact.

---

**Step 7: Diagnose via Okta System Log**

Used the Okta System Log to count and identify direct app assignments:

```
Reports > System Log
Filter: eventType eq "application.user_membership.add"
```

Found 163 direct assignment events — all by Eva Admin during SAML testing.

---

**Step 8: Audit Group State via API**

Created PowerShell API scripts to audit group type, directory linkage, member source, and app assignments:

```powershell
# Check group type and linkage
$result = Invoke-RestMethod `
    -Uri "https://$oktaDomain/api/v1/groups?q=V2_Finance" `
    -Headers $headers

$result | Select-Object id, type, @{N='DN';E={$_.profile.dn}}, @{N='ExternalId';E={$_.profile.externalId}}
```

**Key finding:** All GG_ role groups showed `Type: APP_GROUP` — meaning Okta's app integration had taken ownership, preventing any manual modification via UI or standard API.

---

**Step 9: Remove Direct App Assignments via API**

Since the UI X buttons were grayed out (Okta-mastered), used API to remove all direct assignments:

```powershell
$appIds = @("<gworkspace-app-id>","<sharepoint-app-id>","<slack-app-id>")

foreach ($appId in $appIds) {
    $users = Invoke-RestMethod -Uri "https://$oktaDomain/api/v1/apps/$appId/users" -Headers $headers
    foreach ($user in $users) {
        Invoke-RestMethod -Method DELETE `
            -Uri "https://$oktaDomain/api/v1/apps/$appId/users/$($user.id)" `
            -Headers $headers
        Write-Host "Removed $($user.credentials.userName)"
    }
}
```

---

### Phase 4 — Remediation & Rebuild

**Step 10: Create V2 Groups in AD**

After two sessions fighting the APP_GROUP root cause, the pragmatic decision was made to create fresh groups with a V2 naming convention and build clean from scratch.

```powershell
$groups = @("V2_Finance","V2_Helpdesk","V2_HR","V2_Cloud_Engineer")
foreach ($g in $groups) {
    New-ADGroup -Name $g -GroupScope Global -GroupCategory Security `
        -Path "OU=Groups,OU=_NA,DC=yearwood,DC=local"
}
```

---

**Step 11: Mark GG_ Groups as Deprecated**

Updated the description on all GG_ groups to flag them as inactive without deleting them:

```powershell
$deprecated = @("GG_Finance","GG_Helpdesk","GG_HR","GG_Cloud_Engineer")
foreach ($g in $deprecated) {
    Set-ADGroup -Identity $g -Description "Deprecated"
}
```

> 💡 **Best Practice:** Never delete groups immediately. Mark them, empty them, monitor for 30 days, then delete. Something may still be referencing them.

---

**Step 12: Migrate Users from GG_ to V2 Groups**

```powershell
$groupMap = @{
    "GG_Cloud_Engineer" = "V2_Cloud_Engineer"
    "GG_Finance"        = "V2_Finance"
    "GG_Helpdesk"       = "V2_Helpdesk"
    "GG_HR"             = "V2_HR"
}

# Copy members to V2 groups
foreach ($old in $groupMap.Keys) {
    $new = $groupMap[$old]
    $members = Get-ADGroupMember -Identity $old
    foreach ($member in $members) {
        Add-ADGroupMember -Identity $new -Members $member
        Write-Host "Added $($member.SamAccountName) to $new"
    }
}

# Remove from deprecated GG_ groups
foreach ($old in $groupMap.Keys) {
    $members = Get-ADGroupMember -Identity $old
    foreach ($member in $members) {
        Remove-ADGroupMember -Identity $old -Members $member -Confirm:$false
        Write-Host "Removed $($member.SamAccountName) from $old"
    }
}
```

> 📸 *[Screenshot: Okta Groups page showing V2 groups with correct People counts and AD icon]*
> <img width="1241" height="802" alt="image" src="https://github.com/user-attachments/assets/31cf8710-8c71-428a-9c99-b7be84f014fa" />


---

**Step 13: Run Full Import and Verify**

1. **Directory > Directory Integrations > yearwood.local > Import Now > Full Import**
2. Confirmed all pending assignments
3. Verified each V2 group's **Directories tab** showed `yearwood.local`
4. Ran API audit to confirm AD linkage, correct members, no ghost assignments


---

### Phase 5 — RBAC Implementation & Verification

**Step 14: Assign Groups to Apps via API**

```powershell
# App and Group IDs — retrieve from Okta Admin Console URLs
$gworkspace  = "<gworkspace-app-id>"
$sharepoint  = "<sharepoint-app-id>"
$slack       = "<slack-app-id>"

$finance       = "<v2-finance-group-id>"
$helpdesk      = "<v2-helpdesk-group-id>"
$hr            = "<v2-hr-group-id>"
$cloudEngineer = "<v2-cloudengineer-group-id>"

$assignments = @(
    # Gworkspace — HR and Cloud Engineer only
    @{ appId = $gworkspace; groupId = $hr },
    @{ appId = $gworkspace; groupId = $cloudEngineer },
    # SharePoint — all roles
    @{ appId = $sharepoint; groupId = $finance },
    @{ appId = $sharepoint; groupId = $helpdesk },
    @{ appId = $sharepoint; groupId = $hr },
    @{ appId = $sharepoint; groupId = $cloudEngineer },
    # Slack — all roles
    @{ appId = $slack; groupId = $finance },
    @{ appId = $slack; groupId = $helpdesk },
    @{ appId = $slack; groupId = $hr },
    @{ appId = $slack; groupId = $cloudEngineer }
)

foreach ($entry in $assignments) {
    Invoke-RestMethod -Method PUT `
        -Uri "https://$oktaDomain/api/v1/apps/$($entry.appId)/groups/$($entry.groupId)" `
        -Headers $headers | Out-Null
    Write-Host "Assigned group $($entry.groupId) to app $($entry.appId)" -ForegroundColor Green
}
```

---

**Step 15: Verify with Okta Access Testing Tool**

Navigated to **Reports > Access Testing Tool** and tested a Cloud Engineer user against all three apps:

| Application | Result | Access Via |
|------------|--------|-----------|
| Gworkspace | ✅ Allowed | V2_Cloud_Engineer |
| SharePoint | ✅ Allowed | V2_Cloud_Engineer |
| Slack      | ✅ Allowed | V2_Cloud_Engineer |

> 📸 *[Screenshot: Access Testing Tool showing green Allowed for Gworkspace]*
> <img width="1845" height="836" alt="Screenshot 2026-03-12 at 7 15 57 PM" src="https://github.com/user-attachments/assets/68c0f5f7-6b4f-46e5-b3a0-a961e3291881" />

> 📸 *[Screenshot: Access Testing Tool showing green Allowed for SharePoint]*
> <img width="1389" height="852" alt="Screenshot 2026-03-12 at 7 16 36 PM" src="https://github.com/user-attachments/assets/db5dafd5-4394-46c8-ac9d-68b9bf744546" />

> 📸 *[Screenshot: Access Testing Tool showing green Allowed for Slack]*
> <img width="1370" height="842" alt="Screenshot 2026-03-12 at 7 16 25 PM" src="https://github.com/user-attachments/assets/95b1c205-c223-4682-8a0b-8519e95e00ca" />


Access granted via **group membership** — not direct assignment. RBAC confirmed working.

---

## Key Lessons Learned

### 1. Never Put Application Groups in AD
Application entitlement groups (GG_Slack, GG_VPN, etc.) do not belong in Active Directory. AD's job is identity — who you are. Application access policy belongs in Okta. Creating them in AD and nesting them inside role groups caused Okta to inherit incorrect memberships and store them as Okta-managed records that couldn't be removed through normal means.

### 2. Full Import vs Incremental Import
| Type | Use When | Danger |
|------|----------|--------|
| Full Import | Initial setup, after structural AD changes | Slower but safe |
| Incremental | Stable production environment, picking up daily changes | Only adds, never removes — will lock in bad state if confirmed during misconfiguration |

### 3. Import Confirmation is Permanent
Once you click Confirm on an Okta import, those memberships are treated as intentional. A subsequent Full Import will not re-evaluate them. Get AD right before you ever run an import.

### 4. Okta Data Mastering
Every object in Okta has a master — the authoritative source. If Okta takes ownership (APP_GROUP, Okta-managed), the UI locks you out and even Full Import won't touch it. The only way out is the API.

### 5. Leaver Process — Always Mark Deactivated Users
Deactivated users are not visually obvious. Best practice:
- Move to `DisabledUsers` OU in AD
- Add description with date and reason
- Remove from all role groups immediately
- Prefix with `ZZ_` if your org uses that convention

### 6. Pragmatism Over Perfection
When a root cause has resisted two sessions of investigation, ask: can I achieve the outcome a different way? The V2 workaround delivered working RBAC in 30 minutes. The APP_GROUP root cause is still unknown — the lab is complete.

---

## Tools & Technologies

| Tool | Purpose |
|------|---------|
| Windows Server / ADUC | Domain controller, OU and group management |
| PowerShell | AD administration, Okta API scripting |
| Okta Admin Console | Identity provider configuration |
| Okta AD Agent | Directory integration and sync |
| Okta API | Bulk remediation, audit scripts, group-to-app assignments |
| Okta Access Testing Tool | RBAC verification |
| Okta System Log | Access sprawl forensics |

---

## Project Status

✅ AD domain built and configured  
✅ Okta integration established  
✅ Access sprawl diagnosed and documented  
✅ Direct app assignments removed via API  
✅ V2 role groups created and synced  
✅ RBAC matrix implemented  
✅ Access verified via Okta Access Testing Tool  

⬜ Activate staged users  
⬜ Test negative access (Finance user blocked from Gworkspace)  
⬜ Test joiner-mover-leaver automation  
⬜ Investigate APP_GROUP root cause  

---

*Yearwood.Local IAM Lab — Built as a portfolio project for hands-on identity and access management experience.*
