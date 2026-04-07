
# TryHackMe — Active Directory Basics

![TryHackMe](https://img.shields.io/badge/TryHackMe-Active%20Directory%20Basics-red?style=for-the-badge&logo=tryhackme)
![Status](https://img.shields.io/badge/Status-Completed-brightgreen?style=for-the-badge)

![Completed Active Directory TryHackMe](./completed%20active%20direct%20tryhack.png)
---

## Overview

The main idea behind a Windows domain is to centralise the administration of common components of a Windows computer network in a single repository called **Active Directory (AD)**. The server that runs the Active Directory services is known as a **Domain Controller (DC)**.

---

## Security Groups

| Security Group | Description |
|---|---|
| Domain Admins | Administrative privileges over the entire domain, including all DCs |
| Server Operators | Can administer Domain Controllers but cannot change administrative group memberships |
| Backup Operators | Can access any file regardless of permissions; used for data backups |
| Account Operators | Can create or modify other accounts in the domain |
| Domain Users | Includes all existing user accounts in the domain |
| Domain Computers | Includes all existing computers in the domain |
| Domain Controllers | Includes all existing DCs on the domain |

---

## Organizational Units (OUs)

The domain had an existing OU called **THM** with five child OUs: IT, Management, Marketing, R&D, and Sales — mirroring the company's structure, which allows for efficiently deploying baseline policies per department.

**Default AD Containers:**
- `Builtin` — Default groups available to any Windows host
- `Computers` — Any machine joining the network is placed here by default
- `Domain Controllers` — Default OU containing the DCs
- `Users` — Default users and groups with domain-wide context
- `Managed Service Accounts` — Holds accounts used by Windows services

### Security Groups vs OUs

- **OUs** are used to apply policies to users and computers. A user can only belong to one OU at a time.
- **Security Groups** are used to grant permissions over resources (e.g., shared folders, printers). A user can belong to many groups.

### Tasks Performed

- Disabled Robert and Christine's accounts based on new business changes

![Disabling Accounts in AD](./disable%20acconts.png)

- Deleted a closed department's OU — had to enable **Advanced Features** under the View menu to turn off accidental deletion protection before removing it

![Removing Accidental Deletion Protection](./removed%20accidental%20deletion.png)
---

## Delegation

Active Directory allows you to delegate control of OUs to specific users without making them Domain Admins. This is useful for IT support staff who need to reset passwords but shouldn't have full admin rights.

**What I did:** Delegated control of the Sales OU to Phillip so he could reset passwords for users in that OU only.

![Delegating Control to Phillip](./philip%20can%20change%20password.png)

After resetting Sophie's password as Phillip, I forced a password change at next log on:
```powershell
Set-ADAccountPassword sophie -Reset -NewPassword (Read-Host -AsSecureString -Prompt 'New Password') -Verbose

Set-ADUser -ChangePasswordAtLogon $true -Identity sophie -Verbose
```

![Changing Sophie's Password via PowerShell](./changing%20sophie%20password.png)

---

## Managing Computers in AD

Devices were organized into separate OUs based on their role:

1. **Workstations** — Day-to-day user machines; privileged accounts should never sign into these
2. **Servers** — Provide services to users or other servers
3. **Domain Controllers** — Most sensitive devices; store hashed passwords for all domain accounts

Created two new OUs (`Workstations` and `Servers`) directly under the `thm.local` domain container to properly segregate devices.

---

## Group Policy Objects (GPOs)

GPOs are a collection of settings applied to OUs, targeting either users or computers to enforce a baseline of configurations across the domain.

### Tasks Performed

- **Password Policy** — Configured via: `Computer Configuration → Policies → Windows Settings → Security Settings → Account Policies → Password Policy`
- **Changed Password Length** — Changed password length to be a minimum of 10 characters, linked to the root domain

![Changing Password Length Policy via GPO](./changing%20password%20length.png)

GPOs are distributed via a network share called **SYSVOL**, stored on the DC at `C:\Windows\SYSVOL\sysvol\`. To force an immediate sync:
```powershell
gpupdate /force
```

---

## Authentication Methods

### Kerberos (Default)

1. User sends username + encrypted timestamp to the **Key Distribution Center (KDC)** on the DC
2. KDC responds with a **Ticket Granting Ticket (TGT)** and a Session Key
3. When accessing a service, user sends the TGT + **Service Principal Name (SPN)** to the KDC
4. KDC returns a **Ticket Granting Service (TGS)** encrypted with the Service Owner's hash
5. User presents the TGS to the target service to gain access

The user's password is never sent over the network — only encrypted timestamps and tickets.

### NetNTLM (Legacy)

1. Client sends username to the server
2. Server sends a random **challenge**
3. Client responds with the challenge encrypted using their NTLM password hash
4. Server forwards the challenge + response to the DC for verification
5. DC recalculates and compares — if they match, access is granted

Again, the user's password is never transmitted over the network.

---

## Trees, Forests, and Trusts

### Trees
When two domains share the same namespace (e.g., `thm.local`), they can be joined into a **Tree**. Example: `thm.local` as the root with `uk.thm.local` and `us.thm.local` as subdomains. Each branch has its own DC and Domain Admins managing only their resources.

The **Enterprise Admins** group grants administrative privileges across all domains in the enterprise.

### Forests
When multiple trees with different namespaces are joined together, that is called a **Forest**. Example: THM Inc. merging with MHT Inc. — each company keeps its own domain tree but they exist in the same forest.

### Trust Relationships
Trusts allow users from one domain to be authorized to access resources in another.

- **One-way trust** — Domain AAA trusts Domain BBB: users in BBB can access resources in AAA
- **Two-way trust** — Both domains mutually authorize each other's users (default when joining a tree or forest)

> Note: A trust relationship does not automatically grant access to all resources — it only opens the possibility. What is actually authorized is configured separately.
- **One-way trust** — Domain AAA trusts Domain BBB: users in BBB can access resources in AAA
- **Two-way trust** — Both domains mutually authorize each other's users (default when joining a tree or forest)

> Note: A trust relationship does not automatically grant access to all resources — it only opens the possibility. What is actually authorized is configured separately.
