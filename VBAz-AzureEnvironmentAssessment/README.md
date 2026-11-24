# Azure environment assessment for Veeam backup design

Generates a read-only PowerShell discovery script that inventories every Azure workload Veeam can protect (VMs, Azure SQL, SQL Managed Instance, Cosmos DB, Azure Files, Blob Storage, virtual network configuration, and optionally AKS, Azure VMware Solution and Entra ID). It exports sizing data to CSV for pre-sales sizing and solution design.

What sets it apart from a plain inventory is that **every resource is placed in its environment**: tenant → subscription → region → virtual network → subnet. It also records how the resource is reached (vNet-injected, private endpoint, service endpoint, or public). That is what you need to decide where Veeam worker instances go, how many you need, which subnets have room for them, and which PaaS resources need private connectivity.

| | |
| --- | --- |
| **Product** | Veeam Backup for Microsoft Azure 8 (standalone or as the Veeam Backup & Replication 13 plug-in), plus Veeam Backup & Replication 13 and Veeam Kasten for the optional modules |
| **Interface** | Azure PowerShell (`Az` 12.0 or later, management plane only) |
| **Output** | PowerShell script producing CSV files, an environment summary and a JSON environment map |
| **Tested with** | Claude (2026-11-24). A fresh session given only the two blocks below generated the script. The prompt was then tightened from that session's list of ambiguities, and the script was updated to match (not regenerated from scratch). The result is a single file of about 3,000 lines that parses cleanly, uses no PowerShell-7-only syntax, and passes 36 Pester tests with every Az cmdlet mocked. **Not yet run against a real Azure tenant**, so Az object property names for Cosmos DB, AKS, AVS and the Backup vault cmdlets are unconfirmed. |
| **Author** | [@shapedthought](https://github.com/shapedthought) |
| **Origin** | Expanded from the VM-only [`azure_assessment.ps1`](https://github.com/shapedthought/powershell/blob/master/azure_assessment.ps1) and its [spec](https://github.com/shapedthought/powershell/blob/master/azure_assessment_spec.md) |
| **Generated script** | [`azure_environment_assessment.ps1`](https://github.com/shapedthought/powershell/blob/master/azure_environment_assessment.ps1) and its Pester tests, in shapedthought/powershell |

There are two blocks to paste, in this order:

1. The [workload map](#workload-map). It pins which Azure resource types are in scope and what Veeam product covers each one. Edit it to match what you are selling or designing for.
2. The [prompt](#prompt) itself.

## Workload map

Paste this block **together with** the prompt. It exists so the model does not invent its own idea of what Veeam supports. It also gives you one place to update when support changes. Check it against the [Veeam Backup for Microsoft Azure user guide](https://helpcenter.veeam.com/docs/vbazure/guide/) for the version you are designing for. See the notes on what has and has not been verified.

```text
## Workload map -- treat as ground truth for scope and labelling

Each row: module key | Azure resource type(s) | Veeam protection | design notes
the script must capture. Put the "Veeam protection" text verbatim in the
CandidateVeeamProtection column of that module's CSV. Do not add, drop or
relabel rows.

vm         | Microsoft.Compute/virtualMachines, Microsoft.Compute/disks
           | Veeam Backup for Microsoft Azure: Azure VM
           | Flag per VM: Ultra Disk and Premium SSD v2 (backup supported, but
           | some Veeam Backup & Replication "Restore to Azure" scenarios have
           | limitations with them), ephemeral OS disk,
           | unmanaged (VHD) disks, shared disks, disk encryption type (SSE+PMK,
           | SSE+CMK, ADE, encryption at host), security type (Standard,
           | TrustedLaunch, ConfidentialVM). Report unattached managed disks
           | separately, because they are capacity that no VM policy will cover.

sqldb      | Microsoft.Sql/servers, Microsoft.Sql/servers/databases (single
           | and elastic pool), Microsoft.Sql/servers/elasticPools
           | Veeam Backup for Microsoft Azure: Azure SQL Database
           | Exclude the "master" database. Capture the server's network access
           | (public, firewall rules, "Allow Azure services", private endpoints)
           | and whether Microsoft Entra-only authentication is enforced.

sqlmi      | Microsoft.Sql/managedInstances (+ its databases)
           | Veeam Backup for Microsoft Azure: Azure SQL Managed Instance
           | Always vNet-injected: capture its subnet. Capture whether the
           | public data endpoint is enabled.

cosmos     | Microsoft.DocumentDB/databaseAccounts,
           | Microsoft.DocumentDB/mongoClusters,
           | Microsoft.DBforPostgreSQL/serverGroupsv2
           | API = NoSQL       -> Veeam Backup for Microsoft Azure: Cosmos DB for NoSQL
           | API = MongoDB (RU-based) -> Veeam Backup for Microsoft Azure: Cosmos DB for MongoDB
           | API = MongoDB (vCore, mongoClusters) -> Not supported (inventory only)
           | API = Gremlin     -> Veeam Backup for Microsoft Azure: Cosmos DB for Apache Gremlin
           | API = Table       -> Veeam Backup for Microsoft Azure: Cosmos DB for Table
           | PostgreSQL        -> Veeam Backup for Microsoft Azure: Cosmos DB for PostgreSQL
           | API = Cassandra   -> Not supported (inventory only)
           | Protection blockers (Microsoft.DocumentDB/databaseAccounts only;
           | leave empty for PostgreSQL clusters): Veeam does not process accounts that use
           | periodic backup mode or have multi-region writes enabled (it
           | relies on Azure continuous backup), and cannot restore accounts
           | encrypted with customer-managed keys. Capture backup policy type
           | (Periodic or Continuous, and the continuous tier), multi-region
           | writes and key type for every account, and set ProtectionBlockers
           | accordingly. Only PostgreSQL and MongoDB (RU) accounts can have
           | backups stored in a Veeam repository; the others are protected
           | with Azure continuous backup only. Record this as
           | RepositoryBackupSupported.

files      | Microsoft.Storage/storageAccounts/fileServices/shares
           | Veeam Backup for Microsoft Azure: Azure Files
           | Capture protocol (SMB or NFS), quota, used bytes, access tier, and
           | the storage account's kind, SKU and network rules.

blob       | Microsoft.Storage/storageAccounts (blob service) and containers
           | Veeam Backup & Replication: object storage backup (Azure Blob)
           | Not a native Veeam Backup for Microsoft Azure source workload:
           | that product uses Blob only as a repository target. Capture hierarchical namespace (ADLS Gen2), versioning, soft
           | delete, change feed, point-in-time restore, immutability policy,
           | redundancy and access tier. Many accounts here will turn out to be
           | Veeam repositories, diagnostics or boot-diagnostics accounts:
           | capture enough (name, tags, container count) to tell them apart.

vnet       | Microsoft.Network/virtualNetworks (+ subnets, NSGs, route tables,
           | peerings)
           | Veeam Backup for Microsoft Azure: virtual network configuration backup
           | This module is also the placement backbone for every other module
           | (see the requirements), so it always runs.

aks        | Microsoft.ContainerService/managedClusters    (optional module)
           | Veeam Kasten
           | Capture Kubernetes version, node pools (count, VM size, subnet),
           | network plugin, private cluster flag, and the CSI disk/file drivers.

avs        | Microsoft.AVS/privateClouds                   (optional module)
           | Veeam Backup & Replication: VMware vSphere (Azure VMware Solution)
           | Capture SKU, host count per cluster, management network CIDR, and
           | ExpressRoute / Global Reach connectivity. Do not try to reach
           | vCenter: VM-level inventory comes from a separate vSphere tool.

entra      | Microsoft Entra ID tenant                     (optional module)
           | Veeam Backup & Replication: Microsoft Entra ID
           | A Veeam Backup & Replication plug-in (12.2 or later, 13.x in
           | practice), not a Veeam Backup for Microsoft Azure workload.
           | Counts only: users, groups, applications, service principals,
           | conditional access policies, administrative units. Uses
           | Microsoft.Graph read scopes, not Az. The Graph context must be
           | for the assessed tenant: reconnect with -TenantId if it is not,
           | and skip the module rather than report another tenant's counts.
           | Follow @odata.nextLink so counts are not capped at one page.

other      | Every other resource type returned by Get-AzResource
           | None (gap analysis only)
           | One summary row per resource type per subscription per region with
           | a count. Explicitly list these data services if present, so they
           | are visible as gaps: Azure Database for PostgreSQL flexible server,
           | Azure Database for MySQL flexible server, Azure Cache for Redis,
           | Azure Data Explorer, Azure Synapse, Azure Databricks.
```

## Prompt

```text
I need a single-file PowerShell script that runs a read-only discovery of an
Azure estate ahead of designing a Veeam backup solution. Use the workload map
block I have pasted alongside this prompt as ground truth for which resource
types are in scope and how they are labelled.

The overriding design goal: every resource the script reports must be placed
in its environment -- tenant, subscription, resource group, region,
availability zone, virtual network and subnet -- and must say how it is
reached on the network. Someone designing Veeam worker placement, repository
location and private connectivity must be able to filter or pivot any CSV by
subscription, region and vNet without joining files by hand.

Target: PowerShell 7.2+ and Windows PowerShell 5.1 (so no ternary operator,
no null-coalescing operator, no ForEach-Object -Parallel), Az module 12.0 or
later. It must also run unchanged in Azure Cloud Shell.

Requirements:

1. Read-only. Use management-plane GET operations only (Az cmdlets and
   Azure Monitor metrics). Never list storage account keys, never generate
   SAS tokens, never call data-plane endpoints (no blob listing, no SQL
   queries, no Cosmos DB queries), and never create, modify or tag anything.
   State the minimum roles in the help block: Reader on each subscription
   (includes metrics), Backup Reader for the existing-backup module, and the
   Graph read scopes for the entra module.

2. Parameters:
   -Modules         string[], ValidateSet of the module keys in the workload
                    map plus "all"; default: vm, sqldb, sqlmi, cosmos, files,
                    blob, vnet, other. aks, avs and entra are opt-in.
   -SubscriptionId  string[], optional filter; default is every enabled
                    subscription the account can see.
   -TenantId        optional; passed to Connect-AzAccount.
   -UseDeviceAuthentication  switch, for hosts without a browser.
   -UseExistingContext       switch; skip Connect-AzAccount if a context
                    exists (Cloud Shell).
   -BackupAssessment switch; also collect existing native Azure Backup
                    configuration (requirement 11).
   -SkipMetrics     switch; skip Azure Monitor calls (faster, but used-size
                    columns are left blank).
   -TagNames        string[]; tag keys to emit as extra columns on every
                    resource row (for example Environment, Application,
                    CostCentre). Default none.
   -Anonymise       switch (requirement 13).
   -AnonymiseSalt   optional string (requirement 13).
   -OutputPath      default .\AzureAssessment_<yyyyMMdd_HHmmss>.
   -AcceptDisclaimer switch; skips the interactive consent prompt.
   Use [switch] for every flag, not [bool].

3. Startup checks, in this order, before any Azure call:
   - Az.Accounts present, else exit with the Install-Module command.
   - For each selected module, check its Az sub-modules (Az.Compute,
     Az.Network, Az.Sql, Az.CosmosDB, Az.Storage, Az.Monitor, Az.Resources,
     Az.RecoveryServices, Az.DataProtection, Az.Aks, Az.VMware,
     Microsoft.Graph.Authentication / Microsoft.Graph.Identity.DirectoryManagement).
     A missing Az.Monitor behaves as -SkipMetrics.
     A missing sub-module disables only the modules that need it, with a
     yellow warning and a row in errors.csv. It never aborts the run.
   - Consent prompt: print that this is an unofficial community script, not
     a Veeam tool, with no support, and that it is read-only. Require the
     user to type "yes", unless -AcceptDisclaimer was passed.

4. Collection order (this matters for placement):
   Pass 1 -- for every subscription, collect the network backbone first:
   virtual networks with subnets, private endpoints, NICs and public IPs.
   Pass 2 -- for every subscription, collect each selected workload module.
   Pass 3 -- resolve placement for every workload row against the backbone
   from ALL subscriptions. A private endpoint, NIC or vNet frequently lives
   in a different subscription (hub-and-spoke) from the resource it serves,
   so resolving only within the current subscription is a bug. (A VM, its
   NICs and the NICs' vNet are always in the same subscription; private
   endpoints, vNet rules and AKS/AVS networking are the cross-subscription
   cases.)
   Build hashtable lookups keyed on lower-cased resource ID. Never use
   Where-Object inside a per-resource loop.

5. Placement columns. Every workload CSV row starts with these columns, in
   this order:
     ResourceKey, TenantId, SubscriptionId, SubscriptionName, ResourceGroup,
     Region, Zones, ResourceName, ResourceType, NetworkAccess,
     VNetSubscriptionId, VNetResourceGroup, VNetName, SubnetName,
     SubnetPrefix, PrivateEndpointCount, CandidateVeeamProtection
   followed by one Tag_<key> column per -TagNames entry, then the
   module-specific columns.
   - Child rows (disks, elastic pools, SQL databases, SQL MI databases,
     AKS node pools, vault items) keep their own identity and location
     (ResourceKey, ResourceName, ResourceType, CandidateVeeamProtection,
     TenantId, SubscriptionId, SubscriptionName, ResourceGroup, Region,
     Zones) -- a disk can live in a different resource group from its VM --
     and inherit only the network placement columns (NetworkAccess, VNet*,
     Subnet*, PrivateEndpointCount) from their parent: the attached VM for a
     disk (an unattached disk has blank network columns and NetworkAccess
     N/A), otherwise the parent server, instance, cluster or vault.
     Exception: an AKS node pool with its own subnet is placed on that
     subnet, and inherits the cluster's placement only when it has none.
   - Storage private endpoints are per service (GroupId blob, dfs, file,
     queue, table, web). File shares are placed and counted using only
     'file' endpoints, plus the account's public access and network rules;
     storage_accounts.csv rows (the Blob workload) use only 'blob' and 'dfs'
     endpoints. An account with public access disabled and only a 'blob'
     endpoint therefore has PublicDisabled file shares. FileStorage
     (file-only) accounts are the Azure Files workload: their
     storage_accounts.csv row uses 'file' endpoints and the Azure Files
     CandidateVeeamProtection. List every endpoint in
     network_attachments.csv regardless.
   - Only private endpoint connections in the Approved state (automatic or
     manual) make a resource reachable and count in PrivateEndpointCount.
     Pending and rejected endpoints still count towards their subnet's
     UsedIPs.
   - ResourceKey is the lower-cased Azure resource ID. It is the join key
     across all files, including network_attachments.csv.
   - NetworkAccess is exactly one of, in this precedence order:
       VNetInjected         resource has a NIC or delegated subnet (VMs, SQL MI,
                            AKS node pools, AVS)
       PrivateEndpointOnly  has >=1 private endpoint and public network
                            access is Disabled
       PrivateEndpoint+Public has >=1 private endpoint and public access is
                            still enabled
       PublicDisabled       public access disabled and no private endpoint
                            (unreachable until connectivity is added).
                            Checked before the three public values below,
                            so a disabled endpoint is never reported as
                            reachable through a firewall.
       VNetRules            public endpoint enabled, restricted to selected
                            vNets (service endpoints / virtual network rules)
       PublicFirewall       public endpoint enabled, restricted by IP rules
                            only (including default Deny with zero rules,
                            and SQL servers without an allow-all 0.0.0.0-
                            255.255.255.255 rule)
       Public               public endpoint enabled and open to all networks
       N/A                  resource has no network surface of its own
       Unknown              private endpoint or vNet listing failed in at
                            least one subscription, so an endpoint may exist
                            that the run could not see. Use it instead of a
                            confident value for any resource that could have
                            an endpoint but showed none, and leave its
                            PrivateEndpointCount blank. Record the failed
                            subscriptions in the manifest.
   - The VNet*/Subnet* columns describe the primary attachment: the primary
     NIC for a VM, the delegated subnet for SQL MI, the first node pool
     subnet for AKS, the first private endpoint for PaaS, otherwise the first
     vNet rule. Leave them blank (never "N/A") when there is none.
   - Every attachment -- every NIC of every VM, every private endpoint, every
     vNet rule, every delegated subnet -- also gets one row in
     network_attachments.csv: ResourceKey, AttachmentType (Nic,
     PrivateEndpoint, VNetRule, DelegatedSubnet, NodePool), IsPrimary,
     VNetSubscriptionId, VNetResourceGroup, VNetName, SubnetName,
     SubnetPrefix, PrivateIpAddress, PrivateEndpointName, GroupId (for
     example blob, file, sqlServer, Sql, MongoDB).
   - Use the subnet/vNet IDs to resolve names; do not parse names out of a
     different resource's ID with fixed array offsets.

6. Network backbone outputs (vnet module, always runs):
   virtual_networks.csv: placement columns (NetworkAccess = N/A), address
   spaces, DNS servers, DDoS protection, peering count, peered vNet names
   and peering state, subnet count.
   On vNet and subnet rows the placement VNet*/Subnet* columns describe the
   row itself and PrivateEndpointCount is the number of private endpoints in
   it.
   subnets.csv: placement columns plus SubnetPrefix (all prefixes, joined
   with ";"), PrefixLength, TotalIPs, UsedIPs (distinct IDs across the
   subnet's IP configurations and its private endpoints' NIC IP
   configurations; they overlap, so de-duplicate), AvailableIPs (total minus
   Azure's 5 reserved addresses per prefix minus used),
   NSG name, route table name, NAT gateway, service endpoints (flag
   Microsoft.Storage and Microsoft.Storage.Global explicitly), delegations,
   private endpoint network policies, and a boolean
   CanHostVeeamWorkers that is true when the subnet is not delegated, not
   GatewaySubnet / AzureFirewallSubnet / AzureBastionSubnet /
   RouteServerSubnet, and has AvailableIPs >= 3. IPv4 only for the IP
   counts. Note IPv6 prefixes in a separate column.

7. Module-specific columns (add these after the placement and tag columns).
   Report sizes in GiB, with the column name ending _GiB, rounded to 2 dp.
   - vm (vms.csv): VmSize, vCPUs and Memory_GiB from Get-AzComputeResourceSku
     cached per region, OsType, image publisher/offer/sku, HyperVGeneration,
     SecurityType, PowerState (from one Get-AzVM -Status call per
     subscription, not one per VM), AvailabilitySet, ProximityPlacementGroup,
     DiskCount, OsDiskSku, OsDiskSize_GiB, DataDiskSize_GiB (sum),
     TotalProvisioned_GiB, DiskSkus (distinct, joined ";"), the flags from
     the workload map, NicCount, PrivateIpAddress (primary), HasPublicIp.
     Keep one row per VM. A VM whose NIC cannot be resolved still gets a
     row, with blank network columns and an errors.csv entry.
     disks.csv: every managed disk, with ManagedBy VM name, DiskState,
     Sku, Size_GiB, Tier, Zones, EncryptionType, DiskAccess/network policy,
     IsShared, and Unattached = true when ManagedBy is empty.
   - sqldb (sql_databases.csv): server name, database name, Edition, SKU,
     vCores or DTUs, ElasticPoolName, MaxSize_GiB, Used_GiB (metric),
     Allocated_GiB (metric), BackupStorageRedundancy, PITR retention days,
     LTR weekly/monthly/yearly retention, ZoneRedundant, ReadScale,
     Geo-replication role, server PublicNetworkAccess, firewall rule count,
     AllowAzureServices, EntraOnlyAuth, MinimalTlsVersion.
     sql_elastic_pools.csv: pool, edition, capacity, MaxSize_GiB, database
     count.
   - sqlmi (sql_managed_instances.csv): instance, SKU, vCores, StorageSize_GiB,
     StorageUsed_GiB and ReservedStorage_GiB (metrics), LicenseType, PublicDataEndpointEnabled,
     ProxyOverride, database count; sql_mi_databases.csv: one row per
     database with the instance's placement columns.
   - cosmos (cosmos_accounts.csv): ApiKind (NoSQL, MongoDB, MongoDBvCore,
     Cassandra, Table, Gremlin, PostgreSQL -- derived from resource type,
     Kind and Capabilities, not guessed from the name), BackupPolicyType,
     ContinuousTier, periodic interval and retention, write and read
     regions, MultiRegionWrites, KeyType (service-managed or
     customer-managed), ProtectionBlockers (";"-joined from: PeriodicBackup,
     MultiRegionWrites, CustomerManagedKeyRestore, UnsupportedApi; empty when
     none), RepositoryBackupSupported, per the workload map,
     PublicNetworkAccess, IsVirtualNetworkFilterEnabled, vNet rule count,
     DataUsage_GiB and IndexUsage_GiB (metrics), database/container counts
     where the management plane exposes them, and the
     CandidateVeeamProtection value selected by ApiKind per the workload map.
   - files (file_shares.csv): storage account, share name, Protocol,
     Quota_GiB, Used_GiB (from Get-AzRmStorageShare -GetShareUsage), access
     tier, share soft-delete, snapshot count if available, and the storage
     account's Kind, Sku, LargeFileShares, AzureFilesIdentityBasedAuth.
   - blob (storage_accounts.csv, one row per account, all accounts):
     Kind, Sku/redundancy, AccessTier, HnsEnabled, UsedCapacity_GiB,
     BlobCapacity_GiB, FileCapacity_GiB (metrics), ContainerCount and
     FileShareCount (counted from Get-AzRmStorageContainer and
     Get-AzRmStorageShare, not from metrics), versioning, blob soft delete (days), container soft
     delete, change feed, point-in-time restore, immutability, SFTP, NFSv3,
     AllowSharedKeyAccess, MinimumTlsVersion, PublicNetworkAccess, default
     network action, IP rule count, vNet rule count,
     LikelyPurpose (VeeamRepository, BootDiagnostics, CloudShell, Other)
     inferred only from tags and well-known container/name patterns, and
     say in a comment that it is a heuristic.
   - aks (aks_clusters.csv + aks_node_pools.csv), avs (avs_private_clouds.csv),
     entra (entra_tenant.csv) per the workload map.
   - other (other_resources_summary.csv): every resource type, including
     those in the workload map, with SubscriptionId, SubscriptionName,
     Region, ResourceType, Count, InWorkloadMap (true/false).
   - The mongoClusters and serverGroupsv2 types have no dedicated Az
     cmdlets: read them with Get-AzResource -ResourceType ... -ExpandProperties.

8. Metrics (unless -SkipMetrics). Use Get-AzMetric with an aggregation of
   Maximum (Average for capacity metrics that only expose Average) over the
   last 24 hours at the coarsest grain the metric supports. Use exactly
   these metric names:
     storage account:              UsedCapacity
     .../blobServices/default:     BlobCapacity
     .../fileServices/default:     FileCapacity
     SQL database:                 storage, allocated_data_storage
     SQL managed instance:         storage_space_used_mb, reserved_storage_mb
     Cosmos DB account:            DataUsage, IndexUsage
   If a metric returns nothing or errors, leave the column blank and write
   a row to errors.csv. Do not substitute a different metric, and do not
   fall back to provisioned size in a "used" column.

9. Environment summary outputs:
   environment_summary.csv -- one row per resource SubscriptionId x Region
   x vNet (identified by VNetSubscriptionId, VNetResourceGroup and VNetName,
   since vNet names are not unique; use "(none)" when a resource has no vNet attachment, so public
   PaaS is visible as its own bucket), with: SubscriptionName,
   NetworkAccess mix (e.g. "PrivateEndpointOnly:4;Public:2"), and for each
   module a count column and a capacity column (VMs + provisioned GiB, SQL
   DBs + used GiB, SQL MI + used GiB, Cosmos accounts + data GiB, file
   shares + used GiB, storage accounts + blob GiB, AKS clusters + node count,
   AVS private clouds + host count), plus
   WorkerCapableSubnets (count of subnets in that vNet with
   CanHostVeeamWorkers = true). A capacity total is blank, not 0, when no
   value in the bucket is known (for example with -SkipMetrics), and each
   capacity column has a matching <Module>CapacityComplete column that is
   False when only some values were known.
   region_summary.csv -- one row per Region with the totals, the number of
   distinct subscriptions and vNets holding in-scope resources, and whether
   at least one CanHostVeeamWorkers subnet exists in the region.
   environment_map.json -- a nested document: tenant -> subscriptions ->
   regions -> vNets -> subnets -> resources (ResourceKey, ResourceType,
   ResourceName, NetworkAccess, headline size), with a "(none)" vNet node per
   region for unattached resources. Write it with ConvertTo-Json -Depth 10
   and check that depth is enough for the nesting you build.

10. Resilience:
   - A failing subscription (cannot set context, or 401/403 on the first
     call), module or resource is caught, logged and skipped. A subscription
     is listed as skipped in the manifest only when nothing could be read
     from it. The run always finishes and always writes every CSV header,
     even for an empty result, so downstream tooling does not break.
   - errors.csv: Timestamp, SubscriptionId, Module, ResourceKey, Operation,
     Message. Summarise the error count per module at the end.
   - Retry Azure calls that fail with HTTP 429 or 5xx up to 4 retries (5
     attempts in total) with exponential backoff, via one helper function. Do not retry 403/404.
   - Use -ErrorAction Stop inside try/catch for calls whose failure should
     be logged, rather than SilentlyContinue, so nothing fails silently.
   - Set the subscription context with Set-AzContext -SubscriptionId per
     subscription, and restore the user's original context at the end.
   - Every file write is terminating. Exit with code 1, still writing
     whatever diagnostics can be written, when the run fails before or
     during discovery (sign-in, subscription listing), when any write
     fails, or when anonymisation fails.

11. -BackupAssessment (existing native protection, to show what is already
   covered and what Veeam would replace):
   - Recovery Services vaults: backup items for every workload type the
     vault holds (AzureVM, AzureFiles, MSSQL / SAPHanaDatabase in VM), each
     with ResourceKey of the protected resource where it can be resolved,
     policy name, protection state, last backup status, latest recovery
     point; policies with schedule and retention; jobs for the last 7 days
     only (with -From/-To), not the vault's full history.
   - Backup vaults (Az.DataProtection): backup instances (blobs, disks,
     PostgreSQL, AKS) and their policies.
   - backup_coverage.csv: one row per in-scope workload ResourceKey with
     NativeBackup = RecoveryServicesVault / BackupVault / PlatformBuiltIn
     (the automatic backups of Azure SQL Database, SQL Managed Instance and
     Cosmos DB, when no vault covers them)
     / None / Unknown, the vault name, policy and ProtectionState (so
     stopped protection is visible). Report None only when backup discovery
     succeeded for that subscription; if the backup modules are missing or
     vault or item enumeration failed, report Unknown. If both vault types
     cover a resource, list both, joined with ";".
   Vault files carry the vault's own placement columns, with
   CandidateVeeamProtection left blank.

12. Progress and output: Write-Host with Cyan for progress, Green for
   success, Yellow for warnings and skips; a per-subscription and
   per-module timer; and a final table showing each output file and its
   row count. Write a manifest.json with the script version, start/end
   time, PowerShell and Az versions, parameters used (without the salt),
   the subscriptions processed and skipped, whether anonymisation was on,
   the exit code, BackboneComplete, and the subscriptions whose private
   endpoint listing failed.
   All CSVs: UTF-8, -NoTypeInformation, written once at the end of the run
   from List[object] collections (no += on arrays).

13. -Anonymise:
   - Replace sensitive strings with the first 12 hex characters of
     HMAC-SHA256(salt, value), lower-cased. Lower-case the value before
     hashing for Azure names and IDs, which are case-insensitive, so the
     same resource always hashes the same way across files and the
     placement joins still work.
   - The salt is -AnonymiseSalt if given, otherwise a random 32-byte value
     generated per run. Print a note that two runs can only be correlated if
     the same salt is supplied. Never write the salt to any file or the
     console.
   - Hash: tenant domain names, subscription names, resource group names,
     every resource/vNet/subnet/NIC/private endpoint/server/database/share/
     container/vault/policy name, private and public IP addresses, DNS
     servers, tag values, and ResourceKey. Resource IDs embed names, so
     never emit a raw resource ID anywhere when anonymising.
   - Keep the network shape usable for design: replace each address prefix
     with "<hash>/<prefix length>" so subnet sizes, TotalIPs, AvailableIPs
     and CanHostVeeamWorkers stay intact.
   - Never hash: tenant ID, subscription ID (customers use these to map the
     report back), regions, zones, SKUs, sizes, counts, flags, versions,
     protocol and state values, timestamps, NetworkAccess and
     CandidateVeeamProtection.
   - Output file names must not contain any unhashed name, and nor may
     errors.csv messages or skip reasons. Azure error text embeds names,
     IDs and paths the run may never have seen, so do not try to scrub it:
     replace each message with an allowlisted diagnostic built only from
     the HTTP status code and any Azure error codes from a fixed list in
     the script (for example "HTTP 403; AuthorizationFailed"), or "details
     withheld (anonymised)" when neither is present. Hash the manifest's
     OutputPath as a whole, since it can contain user or customer names.
   - Mask every row, errors.csv included, before writing anything. If
     masking fails, write no data (only a manifest that records the
     failure) and exit 1, so half-masked output can never be labelled
     anonymised. Console progress and warnings must print only masked or
     allowlisted text, and Az/Graph warning streams are suppressed.

14. Structure: a comment-based help block (synopsis, description,
   every parameter, examples for a full run, a Cloud Shell run, a
   single-subscription run with -Modules vm,vnet, and an anonymised run,
   plus the required roles); #Requires -Version 5.1; small helper functions
   (Invoke-WithRetry, Get-Mask, Add-ErrorRow, Get-PlacementColumns,
   Resolve-NetworkAccess, Get-MetricValue); one function per module that
   returns its rows rather than writing files; a main function that
   orchestrates the three passes, the summaries and the export. Keep it a
   single .ps1 file.

Before showing me the result, write Pester 5 tests that mock the Az
cmdlets (no Azure connection) and cover: a VM whose NIC is in a vNet in a
different resource group; a storage account whose private endpoint is in a
hub subscription; a SQL MI on a delegated subnet; a subscription that
returns 403 and is skipped while the run completes; a Cosmos DB account in
periodic backup mode with multi-region writes gets both ProtectionBlockers
while a continuous-mode Gremlin account gets none; a metric that returns nothing
(blank column + errors.csv row); and an anonymised run where the same vNet
name hashes identically in vms.csv, subnets.csv and
network_attachments.csv and no raw resource ID appears in any output file.
Run them and show me the results. Do not connect to a real Azure tenant;
I will run it against one myself.

(If your model cannot execute code, delete the paragraph above and run
those tests yourself.)
```

## What the output looks like

The file set, with the workload CSVs aggregated across all subscriptions (filter on `SubscriptionId`, `Region` and `VNetName` rather than opening one file per subscription):

```text
AzureAssessment_20261124_101500/
  manifest.json                 run metadata
  errors.csv                    everything that was skipped and why
  environment_summary.csv       subscription x region x vNet rollup
  region_summary.csv            per-region totals and worker-subnet availability
  environment_map.json          tenant > subscription > region > vNet > subnet > resource
  subscriptions.csv  tenants.csv  resource_groups.csv
  virtual_networks.csv  subnets.csv  network_attachments.csv
  vms.csv  disks.csv
  sql_databases.csv  sql_elastic_pools.csv
  sql_managed_instances.csv  sql_mi_databases.csv
  cosmos_accounts.csv
  storage_accounts.csv  file_shares.csv
  other_resources_summary.csv
  backup_coverage.csv  rsv_*.csv  backupvault_*.csv     (with -BackupAssessment)
  aks_*.csv  avs_private_clouds.csv  entra_tenant.csv   (opt-in modules)
```

A few illustrative `environment_summary.csv` rows (made-up values) show why the placement columns matter. The public PaaS bucket and the hub vNet holding private endpoints both stand out at a glance:

```text
SubscriptionName,Region,VNetName,NetworkAccessMix,VmCount,VmProvisioned_GiB,SqlDbCount,SqlDbUsed_GiB,FileShareCount,FileShareUsed_GiB,WorkerCapableSubnets
prod-spoke-01,uksouth,vnet-prod-app,VNetInjected:42,42,9216,0,0,0,0,2
prod-hub,uksouth,vnet-hub,PrivateEndpointOnly:11,0,0,6,412.5,5,1830.2,1
prod-spoke-01,uksouth,(none),Public:3;PublicFirewall:2,0,0,2,18.1,3,96.4,0
dev,westeurope,vnet-dev,VNetInjected:17;VNetRules:1,17,2210,0,0,1,12.0,0
```

## Notes

* **Workload map checked 2026-11-24 against the current (late 2026) Veeam Help Center**, not against a live deployment. What was checked:
  * Cosmos DB: supported APIs are NoSQL, MongoDB (RU-based), Apache Gremlin, Table and PostgreSQL. Accounts with periodic backup or multi-region writes are not processed, and customer-managed-key accounts cannot be restored. Sources: [Considerations and Limitations](https://helpcenter.veeam.com/docs/vbazure/guide/limitations.html) and [Performing Cosmos DB Backup](https://helpcenter.veeam.com/docs/vbazure/guide/performing_cosmos_db_backup.html).
  * Cosmos DB repository backups: only the PostgreSQL and MongoDB accounts can have backups stored in a Veeam repository. This comes from the same pages plus an older [R&D forum thread](https://forums.veeam.com/microsoft-azure-f59/comosdb-nosql-t95744.html) and may have changed, so check the release notes.
  * Entra ID: a Veeam Backup & Replication plug-in since 12.2 ([VBR 13 Entra ID overview](https://helpcenter.veeam.com/archive/vbr/13.0.2/userguide/entra_id_overview.html)).

  Still unverified:
  * vCore-based Cosmos DB for MongoDB (`mongoClusters`) is marked unsupported only because the docs say "RU-based".
  * Premium SSD v2 and Ultra Disk backup support and the related restore limitations come from a review comment, not a page I could read.

  When something changes, fix the block, not the prompt, and update this note with the version and date you checked against.
* Run it against one small subscription with `-Modules vm,vnet` first. Check that the vNet and subnet columns in `vms.csv` match what the portal shows before trusting the full run.
* The script is read-only by design. If the generated code calls `Get-AzStorageAccountKey`, `New-AzStorageContext`, `New-AzStorageAccountSASToken`, `Invoke-Sqlcmd`, or any `Set-`, `New-`, `Update-` or `Remove-` cmdlet, reject it. Data-plane access is not needed for sizing, and it would need far more than Reader.
* Hub-and-spoke is the common case, not the edge case: private endpoints and their DNS zones often live in a connectivity subscription. If the account running the script cannot read that subscription, PaaS rows will show `PrivateEndpoint*` access with blank vNet columns, and `errors.csv` will say why. Ask for Reader on the hub too.
* **Open design choice:** `NetworkAccess = Unknown` is used when private endpoint or vNet listing *failed* in a subscription the run could otherwise read. A subscription skipped entirely with 403 does **not** trigger it. That subscription's private endpoints are just as invisible, but counting it would turn every resource with no visible endpoint into `Unknown` whenever one subscription in the estate is unreadable. If your estate keeps private endpoints in a hub subscription, make sure the run can read the hub, and check `errors.csv` for skipped subscriptions before trusting `PublicDisabled`.
* Worker sizing uses `AvailableIPs` and `CanHostVeeamWorkers`. Worker instances need free addresses in a subnet in each region they process, and the usable count is the prefix size minus 5 Azure-reserved addresses minus existing IP configurations. Check the current Veeam Backup for Microsoft Azure guide for the exact per-region requirement and the service endpoint settings for private deployments.
* "Used" columns come from Azure Monitor metrics, which can lag by a few hours and are missing for resources that have just been created. Blank means unknown, not zero. With `-SkipMetrics`, sizing falls back to provisioned or quota values only, and those columns are labelled as such.
* Anonymised output is linkable within a run, and across runs only if you pass the same `-AnonymiseSalt`. The original script used an unsalted 8-character SHA-256 hash. An unsalted hash of an IPv4 address can be reversed by brute force in seconds, which is why this version uses a keyed HMAC and keeps subnet prefix lengths in clear.
* Subscription and tenant IDs are deliberately left in clear so the customer can map the report back. Remove them yourself if the report leaves the organisation.
* For large estates (hundreds of subscriptions), per-subscription cmdlets are slow. A sensible follow-up prompt is to move pass 1 and the `other` module onto Azure Resource Graph (`Search-AzGraph`, with paging past 1,000 rows), keeping metrics and the backup module on cmdlets.
* The test run could not check Az object property names against real objects. The ones to check first on a live run: the Cosmos DB continuous backup tier, AKS storage profile (CSI driver) fields, AVS cluster and ExpressRoute fields, the nested `Property` objects returned by the Az.DataProtection cmdlets, and whether `Get-AzPrivateEndpoint` and `Get-AzCosmosDBAccount` work without `-ResourceGroupName` in your Az version. A wrong property name shows up as a blank column, not an error, so compare a few rows with the portal.
* Windows PowerShell 5.1 silently truncates `ConvertTo-Json` output deeper than `-Depth`, where PowerShell 7 warns. If `environment_map.json` looks shallow on 5.1, that is why.
* Models tend to get these wrong: parsing vNet names out of subnet IDs with `Split('/')[-3]` (fine for NICs, wrong for anything else), calling `Get-AzVM -Status` once per VM, resolving private endpoints only within the current subscription, using `ConvertTo-Json` with the default depth of 2, and counting `master` as a SQL database. The requirements call each of these out, so check them in the output.

## Disclaimer

See the [read this first](../README.md#%EF%B8%8F-read-this-first) section in the root README. This prompt is a starting point, not a finished solution. Review and test all generated output before running it against a production tenant, and agree with the customer what data leaves their environment before sharing the output.
