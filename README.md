# modern-Paas-securesecrets

# 🔑 Lab 03 — Modernizing to PaaS & Securing Secrets

This lab moves your infrastructure from self-managed to fully managed. You will decommission the database VM from Lab 02 and replace it with **Azure SQL Database** — a managed PaaS service where Microsoft handles patching, backups, and availability. You will also introduce **Azure Key Vault** to store your database password securely, and use a **Managed Identity** on your web server so it can retrieve that password at runtime — without a password ever being hardcoded anywhere.

---

## 📐 Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        rg-lab03-[yourname]  ·  East US                      │
│                                                                             │
│   ┌─────────────────────┐    1. fetch secret    ┌────────────────────────┐  │
│   │    vm-web-01         │ ─────────────────────▶│   Azure Key Vault      │  │
│   │    (from Lab 02)     │                       │   kv-lab03-[yourname]  │  │
│   │                      │                       │                        │  │
│   │    Managed Identity  │                       │   Secret:              │  │
│   │    system-assigned   │                       │   SqlAdminPassword     │  │
│   │                      │                       │                        │  │
│   │                      │  2. connect with      │   Role assigned:       │  │
│   │                      │     retrieved password│   Key Vault Secrets    │  │
│   │                      │ ──────────────────┐   │   User                 │  │
│   └─────────────────────┘                   │   └────────────────────────┘  │
│                                             │                               │
│   ┌─────────────────────┐                   │   ┌────────────────────────┐  │
│   │    vm-db-01          │                   └──▶│   Azure SQL Database   │  │
│   │    ✖ DECOMMISSIONED  │                       │   sqldb-app            │  │
│   │    replaced by PaaS  │                       │   Basic · 5 DTUs       │  │
│   └─────────────────────┘                       └────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Secret retrieval flow:**
```
vm-web-01 (Managed Identity)  ──▶  Key Vault (fetch SqlAdminPassword)  ──▶  Azure SQL Database (connect)
```

> **Why this matters:** At no point is the password stored in code, a config file, or on the server. The VM proves its identity to Key Vault automatically, retrieves the password at runtime, and uses it to connect to SQL. This is a production-grade secrets management pattern used across the industry.

---

## 🎯 Objectives

By the end of this lab you will have:

- Decommissioned `vm-db-01` from Lab 02, simulating a real-world database migration cutover
- Deployed an **Azure SQL Database** — a fully managed PaaS relational database
- Deployed an **Azure Key Vault** and stored the SQL admin password as a secret
- Enabled a **System-assigned Managed Identity** on `vm-web-01`
- Granted the VM's identity read access to Key Vault secrets using **RBAC**
- Confirmed the database is live using **Azure Monitor Metrics**

---

## 🧠 Key Concepts Before You Start

**What is "refactoring" in cloud infrastructure?**
Refactoring means improving how something is built without changing what it does. Here, you are swapping a self-managed database VM (where you are responsible for OS updates, backups, and availability) for a managed Azure service (where Microsoft handles all of that automatically). This is a common real-world migration pattern called "lift and modernize."

**PaaS vs IaaS:**
In Lab 02 you used IaaS — you deployed a full virtual machine and were responsible for everything on it. In this lab, Azure SQL Database is PaaS: you use the database without managing any underlying server, operating system, or patching. You just create the database and start using it.

---

## ✅ Prerequisites

| Requirement | Details |
|---|---|
| Active Azure Subscription | Must be able to log in to [portal.azure.com](https://portal.azure.com) |
| Completed Lab 02 | `vm-web-01` must be running in `rg-lab02-[yourname]` |
| Completed Week 3 Video Modules | Required before starting this lab |

**How to verify Lab 02 is complete:**

1. Go to [portal.azure.com](https://portal.azure.com)
2. Click **Resource groups** in the left menu
3. Open `rg-lab02-[yourname]`
4. Confirm `vm-web-01` is listed with a **Status** of `Running`

If `vm-web-01` is stopped, click on it, then click **Start** and wait for it to reach the `Running` state before proceeding.

---

## 🏷️ Naming Convention Reference

Replace `[yourname]` with your actual name or initials. Keeping names consistent across labs ensures resources are easy to identify.

| Resource | Name | Notes |
|---|---|---|
| Resource Group | `rg-lab03-[yourname]` | e.g., `rg-lab03-jcharles` |
| SQL Server | `sql-server-[yourname]` | Must be globally unique, all lowercase |
| SQL Database | `sqldb-app` | Exact name — do not change this |
| Key Vault | `kv-lab03-[yourname]` | Must be globally unique |
| Region | East US | Use the nearest US region if East US is unavailable |

> 💡 **Why globally unique names?** SQL Servers and Key Vaults receive public DNS hostnames (e.g., `sql-server-jcharles.database.windows.net`). Because these names are public, no two resources across all of Azure can share the same name — the same reason two websites cannot share a domain name.

---

## 🛠️ Step-by-Step Instructions

### Phase 1 — Decommission the Database VM

In a real migration, you would migrate data to the new platform first, verify everything works, then shut down the old server. This phase simulates that final decommission step.

> 💡 **Why delete instead of stop?** A stopped VM in Azure still incurs charges for its managed disk and reserved IP address. Fully deleting the VM and its associated resources is correct practice when a resource is no longer needed.

1. In the Azure Portal, click **Resource groups** in the left-hand navigation menu
2. Click `rg-lab02-[yourname]`
3. Locate the three resources associated with `vm-db-01`:
   - The virtual machine: `vm-db-01`
   - Its OS disk (named something like `vm-db-01_OsDisk_...`)
   - Its network interface (named something like `vm-db-01-nic` or `vm-db-01VMNic`)
4. Click the checkbox next to each of those three resources to select them
5. Click **Delete** at the top of the resource list
6. Type `delete` in the confirmation box and click **Delete**

> ⚠️ **Critical:** Do NOT delete the Resource Group or `vm-web-01`. You only want to remove the three resources tied to `vm-db-01`. If you are unsure which disk or NIC belongs to the DB VM, click each resource and check the **Overview** tab — it will show the associated VM name.

**Verify:** Refresh the resource group. `vm-db-01`, its disk, and its NIC should no longer appear. `vm-web-01` and its resources should still be present.

---

### Phase 2 — Deploy Azure SQL Database (PaaS)

**What is Azure SQL Database?**
Azure SQL Database is a fully managed relational database service built on Microsoft SQL Server. Unlike running SQL Server on a VM, you do not manage the underlying operating system, apply patches, or configure backups — Azure handles all of that. You simply create the database and use it.

**SQL Server vs SQL Database in Azure:**
A SQL Server is a logical management container — it holds settings like the admin login, firewall rules, and region. A SQL Database is the actual database that lives inside that server. You must create the server first, then create databases inside it.

**Steps:**

1. In the top search bar, type `SQL databases` and click the result
   - Select **SQL databases** (the PaaS service)
   - Do not select **SQL servers** — that is a management container you will configure as part of the database creation wizard

2. Click the **dropdown arrow** on the **+ Create** button and select **SQL database** (the standard option)
   - Do not select **SQL database (Free offer)** — it does not support the Basic DTU pricing tier this lab uses

**Basics tab:**

> ⚠️ **Remove the Free offer first.** If a green "Free offer applied" banner appears at the top, click **Remove offer** or **Advanced configuration** inside that banner and wait for it to disappear completely. The DTU-based Basic tier is only available after the free offer has been fully removed.

3. **Subscription** — Leave as your default subscription
4. **Resource group** — Click **Create new**, enter `rg-lab03-[yourname]`, click **OK**
5. **Database name** — Clear any auto-filled value and enter `sqldb-app`

   > ⚠️ Common mistake: Do not enter the server name here. The database name and server name are two separate fields. Using `sqldb-app` exactly is important — later phases reference it by name.

6. **Server** — Click **Create new**. A panel will open on the right. Fill it in as follows:

   | Field | Value |
   |---|---|
   | Server name | `sql-server-[yourname]` (all lowercase, no spaces) |
   | Location | `(US) East US` |
   | Authentication method | Use SQL authentication |
   | Server admin login | `sqladmin` |
   | Password | Create a strong password and write it down — you will need it in Phase 4 |
   | Confirm password | Re-enter the same password |

   > 💡 **Password requirements:** Minimum 8 characters; must include uppercase, lowercase, a number, and a special character. Example: `Lab03@Secure!`

   > 💡 **About the authentication options:** Select **Use SQL authentication** only. The "Use both" option adds unnecessary Microsoft Entra complexity for this lab. "Use Microsoft Entra-only authentication" disables username/password login entirely, which would break this lab.

   Click **OK** to close the panel and return to the main form.

7. **Want to use SQL elastic pool?** — Select **No**
8. **Workload environment** — Select **Development**

   > ⚠️ This field defaults to Production, which selects the Hyperscale tier at $320+/month. Selecting Development sets a lower-cost starting configuration and makes the Basic DTU tier available.

9. Under **Compute + storage**, click **Configure database**

   > If you do not see a DTU option inside Configure database, the free offer is still active. Return to the top of the Basics tab, click **Remove offer**, wait for the page to reload, then click **Configure database** again.

10. At the top of the configuration panel, select **DTU-based** as the purchasing model
11. Select **Basic** from the service tier options
12. Confirm the cost summary shows approximately **$4.99/month**
13. Click **Apply**

**Networking tab:**

14. Click the **Networking** tab
15. **Connectivity method** — Change from `No access` to **Public endpoint**
16. Once Public endpoint is selected, set the following toggles:
    - **Allow Azure services and resources to access this server** — Set to **Yes**
    - **Add current client IP address** — Set to **Yes**
17. **Connection policy** — Leave as `Default`
18. **Minimum TLS version** — Leave as `TLS 1.2`

**Security tab:**

19. Click the **Security** tab
20. **Enable Microsoft Defender for SQL** — Select **Not now**

   > Microsoft Defender for SQL costs $15/server/month and is valuable in production but unnecessary for this lab.

**Additional Settings tab:**

21. Click the **Additional settings** tab
22. **Use existing data** — Leave as `None`
23. **Collation** — Leave as the default `SQL_Latin1_General_CP1_CI_AS`
24. **Maintenance window** — Leave as `System default (5pm to 8am)`

**Review + create:**

25. Click **Review + create** and confirm the summary shows:
    - Tier: **Basic**
    - Estimated cost: approximately **$4.99/month** (or $0.00 if covered by free credits)
    - Server name: `sql-server-[yourname]`
    - Resource group: `rg-lab03-[yourname]`

26. Click **Create** and wait approximately 2–3 minutes for deployment

**Verify:** Click **Go to resource**. The database Overview page should show the server name, database name `sqldb-app`, and status `Online`.

---

### Phase 3 — Deploy Azure Key Vault

**What is Azure Key Vault?**
Azure Key Vault is a cloud service for securely storing and accessing secrets — passwords, API keys, certificates, and connection strings. Instead of storing a password in a config file or hardcoding it in an application, you store it in Key Vault and your application retrieves it at runtime. The password is never written anywhere it could be accidentally exposed or committed to source control.

**Steps:**

1. In the top search bar, type `Key vaults` and click the result
2. Click **+ Create**

**Basics tab:**

3. **Subscription** — Leave as your default subscription
4. **Resource group** — Select `rg-lab03-[yourname]`
5. **Key vault name** — Enter `kv-lab03-[yourname]`
6. **Region** — Select `(US) East US`
7. **Pricing tier** — Confirm it is set to **Standard**

   > **Standard vs Premium:** Standard covers secrets, keys, and certificates — everything needed for this lab. Premium adds Hardware Security Module (HSM) support used in high-security production environments.

**Recovery options:**

| Setting | Value | Action |
|---|---|---|
| Soft-delete | Enabled (cannot be changed) | Leave as-is |
| Days to retain deleted vaults | 90 | Leave as 90 |
| Purge protection | Disable purge protection | Leave disabled |

> ⚠️ **Important for cleanup:** Leave Purge protection **disabled**. Enabling it will prevent you from permanently deleting the Key Vault at the end of the lab until a 90-day retention period expires.

**Access configuration tab:**

8. Click the **Access configuration** tab
9. **Permission model** — Leave as **Azure role-based access control (RBAC)**

   > RBAC is Azure's central permission management system. Instead of configuring permissions directly on the Key Vault, you assign a standardized role — such as "Key Vault Secrets User" — to an identity. This is consistent across all Azure services and easier to audit.

10. Leave all three **Resource access** checkboxes unchecked — none of those scenarios apply to this lab

**Networking tab:**

11. Click the **Networking** tab
12. Confirm the following — no changes are needed:
    - **Enable public access** — Checked
    - **Allow access from** — All networks

   > Note: The portal displays a warning about public network access. This is expected and acceptable for a lab. Access is still protected by RBAC role assignments — no one can read secrets without the correct role assigned.

**Review + create:**

13. Click **Review + create**, confirm the summary, then click **Create**

**Verify:** Click **Go to resource**. You should see your Key Vault Overview page with the Vault URI (e.g., `https://kv-lab03-[yourname].vault.azure.net/`).

---

#### Grant Yourself Access to Key Vault

> ⚠️ **Required before Phase 4:** With RBAC as the permission model, no one has access to the Key Vault by default — including the person who created it. If you navigate to Secrets right now, you will see: "The operation is not allowed by RBAC... You are unauthorized to view these contents." This is expected. You must assign yourself the **Key Vault Administrator** role before you can create secrets.

1. In your Key Vault resource, click **Access control (IAM)** in the left-hand menu
2. Click **+ Add** → **Add role assignment**
3. In the **Role** tab, search for `Key Vault Administrator`, select it, and click **Next**
4. In the **Members** tab, set **Assign access to** → **User, group, or service principal**
5. Click **+ Select members**, search for your own Azure account email address, select it, and click **Select**
6. Click **Review + assign**, then **Review + assign** again to confirm

Wait 1–2 minutes for the role assignment to propagate, then navigate to **Objects → Secrets**. The unauthorized message should be gone.

> 💡 **Why Key Vault Administrator for yourself but Key Vault Secrets User for the VM?** Key Vault Administrator gives full management access — create, read, update, and delete. You need this as the person setting up the vault. Key Vault Secrets User (assigned to the VM in Phase 5) grants read-only access to secret values — the minimum permission the VM needs to retrieve the password at runtime. This is the principle of least privilege applied in practice.

---

### Phase 4 — Store the SQL Password as a Secret

**Why store it in Key Vault instead of a config file?**
In a real application, your web server's code needs to connect to the database. If you hardcode the password in the application or store it in a config file on the server, it becomes a security risk. Config files can be accidentally exposed, committed to source control, or read by anyone with server access. Key Vault removes the password from everywhere except one secure, audited location.

**Steps:**

1. In your Key Vault resource, click **Secrets** under the **Objects** section in the left-hand menu
2. Click **+ Generate/Import**
3. Fill in the form:

   | Field | Value |
   |---|---|
   | Upload options | Manual |
   | Name | `SqlAdminPassword` |
   | Secret value | The SQL admin password you created in Phase 2 |
   | Content type | Leave blank |
   | Set activation date | Leave unchecked |
   | Set expiration date | Leave unchecked |
   | Enabled | Yes |

   > **Secret name rules:** Key Vault secret names can only contain letters, numbers, and hyphens. No spaces or special characters in the name itself.

4. Click **Create**

**Verify:** `SqlAdminPassword` should now appear in the Secrets list with a status of **Enabled**. Click on it, then click the current version to confirm it was saved. Key Vault intentionally hides the secret value after storage — you will see metadata but not the password itself.

---

### Phase 5 — Enable Managed Identity & Grant Key Vault Access

This phase has two parts. First you enable a Managed Identity on the web VM, then you grant that identity permission to read secrets from Key Vault.

**What is a Managed Identity?**
A Managed Identity is an automatically managed identity in Azure Active Directory that is assigned directly to an Azure resource. Instead of creating a username and password for your VM to authenticate with other services, Azure manages a secure credential behind the scenes. The VM can prove "I am `vm-web-01`" to other Azure services without any password being stored anywhere.

Think of it like a building access badge permanently attached to a person — they do not need to remember a PIN; the badge proves who they are automatically.

---

#### Part A — Enable Managed Identity on the Web VM

1. In the Azure Portal, navigate to **Resource groups** → `rg-lab02-[yourname]`
2. Click on `vm-web-01`
3. In the left-hand menu, find **Identity** — either scroll to the **Security** section or type `identity` in the left menu search box to jump directly to it
4. Stay on the **System assigned** tab
5. Change the **Status** toggle from **Off** to **On**
6. Click **Save**
7. A confirmation dialog will appear explaining that a system-assigned identity will be registered in Azure AD — click **Yes**

**Verify:** After saving, the page displays an **Object (principal) ID** — a unique identifier for your VM's identity in Azure AD. Copy this value and keep it handy.

---

#### Part B — Grant the VM Access to Key Vault via RBAC

1. Navigate to your Key Vault `kv-lab03-[yourname]`
2. In the left-hand menu, click **Access control (IAM)**
3. Click **+ Add** → **Add role assignment**

**Role tab:**

4. Search for `Key Vault Secrets User`
5. Select **Key Vault Secrets User** from the results
6. Click **Next**

   > **Why this role specifically?** Key Vault Secrets User grants permission to read secret values — nothing more. It cannot create, update, or delete secrets. This is the principle of least privilege: grant only the minimum access required for the task.

**Members tab:**

7. Set **Assign access to** → **Managed identity**
8. Click **+ Select members**
9. In the panel that opens:
   - **Subscription** — Leave as your default subscription
   - **Managed identity** — Select **Virtual machine** from the dropdown
   - `vm-web-01` should appear in the list — click on it to select it
10. Click **Select** to confirm
11. Click **Next**

   > If `vm-web-01` does not appear in the list, return to Part A and confirm you clicked **Save** and that an Object (principal) ID was generated. Wait 60 seconds for the identity to propagate and try again.

**Review + assign:**

12. Confirm the summary shows:

    | Field | Expected Value |
    |---|---|
    | Role | Key Vault Secrets User |
    | Scope | Contains `kv-lab03-[yourname]` in the path |
    | Member name | `vm-web-01` |
    | Member type | Virtual machine |

13. Click **Review + assign**, then **Review + assign** again to confirm

**Verify:** Click the **Role assignments** tab on the Access control (IAM) page. You should see `vm-web-01` listed with the role `Key Vault Secrets User` and type `VirtualMachines`.

---

### Phase 6 — Validate with Azure Monitor

**What is observability?**
Observability is the ability to understand the internal state of a system by examining its outputs — metrics, logs, and traces. In cloud infrastructure, setting up monitoring is not optional. It is how you know your resources are healthy, how you detect problems before users do, and how you demonstrate to stakeholders that services are running. This phase builds that habit.

**Steps:**

1. In the top search bar, type `SQL databases` and click the result
2. Click `sqldb-app` in the list

   > If you only see the server name (e.g., `sql-server-charles`) and not `sqldb-app`, you are on the SQL Server resource, not the database. Click the server, scroll to **Data management → SQL databases** in the left menu, and click `sqldb-app` from there.
   >
   > How to confirm you are on the database: The page header should read `sqldb-app` with a subtitle of `SQL database`. If it shows the server name with a subtitle of `SQL server`, you are one level too high.

3. Check the **Pricing tier** shown in the Essentials section:
   - If it shows `Basic` → use **DTU percentage** as your metric in the next step
   - If it shows `Free - General Purpose - Serverless` → use **CPU percentage** as your metric instead

4. In the left-hand menu, click **Monitoring** to expand it, then click **Metrics**
5. Configure the chart:
   - **Metric:** Select `DTU percentage` (or `CPU percentage` if on Free tier)
   - **Aggregation:** Select `Max`

6. A chart will render — even if it shows near-zero activity, the line confirms the database is live, reachable, and being monitored by Azure

**What does DTU percentage tell you?**

| DTU % | What it means |
|---|---|
| ~7% | Database is nearly idle |
| ~80% | Database is under heavy load — consider scaling up |
| 100% | Database is maxed out — queries may be queuing or slowing down |

In production you would set up alerts on DTU percentage so you are notified before the database becomes a bottleneck. For this lab, any visible line on the chart confirms the database is live and correctly configured.

**Verify:** A chart appears with a visible line. No error messages are displayed on the Metrics page.

---

## 🔧 Troubleshooting

| Issue | Likely Cause | Resolution |
|---|---|---|
| SQL Database creation fails | Server name already taken by another Azure user | Append your birth year to the name, e.g., `sql-server-jcharles1990` |
| Database shows Free/Serverless tier after deployment | Free offer was not removed before creating the database | Do not attempt to change the tier after deployment — the vCore options shown can cost $400+/month if applied accidentally. Proceed as-is and use **CPU percentage** in Phase 6 instead of DTU percentage. The observability concept is identical |
| `vm-web-01` does not appear when selecting the Managed Identity | Managed Identity was not saved or has not propagated | Return to `vm-web-01` → **Identity** and confirm an Object ID is present. Wait 60 seconds and try again |
| "You are unauthorized to view these contents" on the Key Vault Secrets page | Your own user account has not been assigned an RBAC role on the Key Vault | Go to Key Vault → **Access control (IAM)** → Add role assignment → assign yourself **Key Vault Administrator**. Wait 1–2 minutes and refresh |
| Metrics chart shows no data | Newly created database may take a few minutes to emit metrics | Wait 5 minutes and refresh the Metrics page |
| Database was created with the wrong name | Database name field was not cleared before creation | Azure does not support renaming databases. Delete the incorrectly named database, then go to SQL databases → Create → SQL database. Select the existing server from the dropdown (do not create a new one), enter `sqldb-app` as the database name, and complete the wizard |

---

## 📚 Key Concepts Learned

| Term | Definition |
|---|---|
| **PaaS** | Platform as a Service — a managed cloud service where the provider handles the underlying infrastructure |
| **IaaS** | Infrastructure as a Service — you manage the VM and everything on it |
| **Azure SQL Database** | A fully managed relational database service based on Microsoft SQL Server |
| **Azure Key Vault** | A secure store for secrets, keys, and certificates |
| **Secret** | A sensitive value (such as a password or API key) stored securely in Key Vault |
| **Managed Identity** | An Azure AD identity automatically assigned to an Azure resource — no password required |
| **RBAC Role Assignment** | Granting an identity permission to perform actions on an Azure resource by assigning a standardized role |
| **Key Vault Secrets User** | A built-in Azure role that grants read-only access to Key Vault secret values |
| **Least Privilege** | The security principle of granting only the minimum permissions needed to perform a task |
| **Observability** | The practice of monitoring metrics, logs, and traces to understand system health |
| **DTU** | Database Transaction Unit — a blended measure of compute, memory, and I/O capacity for Azure SQL |
| **LRS** | Locally-redundant storage — data is replicated within a single datacenter |

---

## ✅ Lab Completion Checklist

- [ ] `vm-db-01`, its OS disk, and its NIC have been deleted from `rg-lab02-[yourname]`
- [ ] `vm-web-01` is still running in `rg-lab02-[yourname]`
- [ ] Azure SQL Database `sqldb-app` deployed on the Basic tier in `rg-lab03-[yourname]`
- [ ] Azure Key Vault `kv-lab03-[yourname]` deployed in `rg-lab03-[yourname]`
- [ ] Secret `SqlAdminPassword` created and **Enabled** in Key Vault
- [ ] System-assigned Managed Identity **enabled** on `vm-web-01` with an Object ID present
- [ ] Key Vault role assignment created for `vm-web-01` with the **Key Vault Secrets User** role
- [ ] Azure Monitor Metrics chart confirmed showing DTU percentage (or CPU percentage) for `sqldb-app`

---

## 🧹 Clean Up Resources

> **Always clean up after completing a lab** to avoid unexpected charges.

**Delete the Lab 03 resource group:**

1. Go to **Resource groups** → `rg-lab03-[yourname]`
2. Click **Delete resource group**
3. Type the resource group name to confirm, then click **Delete**

This removes the SQL Server, SQL Database, and Key Vault in a single operation.

**Optionally delete the Lab 02 resource group:**

If you have finished with `vm-web-01` and all Lab 02 content, repeat the steps above for `rg-lab02-[yourname]`.

> ⚠️ Do not delete Lab 02 resources if you plan to continue to Lab 04. Check the Lab 04 prerequisites before cleaning up.

---

## 🔗 References

- [Azure SQL Database Documentation](https://learn.microsoft.com/en-us/azure/azure-sql/database/sql-database-paas-overview)
- [Azure Key Vault Overview](https://learn.microsoft.com/en-us/azure/key-vault/general/overview)
- [Managed Identities for Azure Resources](https://learn.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/overview)
- [Azure RBAC Documentation](https://learn.microsoft.com/en-us/azure/role-based-access-control/overview)
- [Azure Monitor Metrics](https://learn.microsoft.com/en-us/azure/azure-monitor/essentials/data-platform-metrics)

---

*Part of the Azure Cloud Fundamentals Lab Series — Lab 03 of N*
