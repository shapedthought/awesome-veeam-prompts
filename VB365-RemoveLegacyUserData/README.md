# Remove user data from a legacy VB365 repository

Generates a PowerShell script that removes backed-up user data (mailbox, archive mailbox, OneDrive, personal sites) from a Veeam Backup for Microsoft 365 repository, using the supported `Remove-VBOEntityData` cmdlet. The typical case is a customer who has moved to Veeam Data Cloud and wants the legacy object storage (usually Azure Blob) emptied for a list of users, without deleting blobs by hand and leaving the repository metadata inconsistent.

The script is built to fail safe. By default it only reports what it found. Deletion needs an explicit mode and a typed confirmation, and it stops on anything ambiguous rather than guessing which user was meant.

| | |
| --- | --- |
| **Product** | Veeam Backup for Microsoft 365 8 |
| **Interface** | PowerShell (`Veeam.Archiver.PowerShell` module, run on the VB365 server) |
| **Output** | PowerShell script, CSV results file, transcript |
| **Tested with** | Not yet. The cmdlet contract was checked against the Help Center pages (dated 2026-08-21) on 2026-11-27. It has not yet been run through an LLM or against a live VB365 server. |
| **Author** | [@shapedthought](https://github.com/shapedthought) |

There are two blocks to paste, in this order:

1. The [cmdlet contract](#cmdlet-contract). It pins the parameter sets the model must use and lists what is not documented, so the model checks it at run time instead of inventing it.
2. The [prompt](#prompt) itself.

## Cmdlet contract

Paste this block **together with** the prompt. The model cannot reach the customer's VB365 server, so without it a wrong parameter or property name is only discovered on the run that deletes data.

```text
## Cmdlet contract -- treat as ground truth

Checked against the Veeam Backup for Microsoft 365 8 PowerShell reference.
Use exactly these parameter sets. Do NOT invent parameters, and do NOT guess
property names marked UNVERIFIED below.

Get-VBORepository
  Get-VBORepository -Name <String>             primary (backup job) repository
  Get-VBORepository -Name <String> -LongTerm   backup copy / archive repository

Get-VBOEntityData   (read-only)
  ByType (default):
    Get-VBOEntityData -Repository <VBORepository> -Type User [-Name <String>]
                      [-Organization <VBOOrganization>]
  ByUser:
    Get-VBOEntityData -Repository <VBORepository> -User <VBOOrganizationUser>
    (-User takes a LIVE organization user from Get-VBOOrganizationUser. It
     takes no -Type, -Name or -Organization.)
  -Name is documented only as "data with the specified name". Wildcard
  support and match semantics are UNDOCUMENTED. A name can match more than
  one user, and the result is then an array, which Remove-VBOEntityData
  rejects with "Cannot convert 'System.Object[]' to the type ...VBOUserData".
  Output type is not named in the docs. Property names on the returned
  object (display name, email/UPN, per-workload flags) are UNVERIFIED.

Get-VBOOrganizationUser
  Get-VBOOrganizationUser -Organization <VBOOrganization> -UserName <UPN>
  Queries the LIVE tenant. Users already deleted from Entra ID are NOT
  returned, even though their backed-up data still exists.

Remove-VBOEntityData   (DESTRUCTIVE, irreversible)
  Remove-VBOEntityData -Repository <VBORepository> -User <VBOUserData>
                       [-Mailbox] [-ArchiveMailbox] [-OneDrive] [-Sites]
                       [-RunAsync] [-WhatIf] [-Confirm]
  -User must be exactly ONE VBOUserData object from Get-VBOEntityData.
  -Sites is the user's sites data. Group and Team data are separate
  parameter sets (-Group, -Team) and are OUT OF SCOPE.
  -Confirm defaults to True. -WhatIf is supported.
  Restriction: data cannot be removed from object storage repositories
  with data immutability enabled.
  Works on backup copy target repositories (use -LongTerm to get them).
  Behaviour while a job is running against the repository is UNDOCUMENTED.

Get-VBODataManagementSession   (read-only)
  Get-VBODataManagementSession
  Returns data management sessions. Property names are UNVERIFIED.

Never call any other Remove-, Set-, New-, Start-, Enable- or Disable- cmdlet.
Never touch the object storage (Azure Blob, S3) directly.

For every UNVERIFIED item: at run time, inspect the real object with
Get-Member, use the property only if it exists, and otherwise STOP with a
message that names the missing property and prints the available ones. Do
not fall back to a similar-looking name.
```

## Prompt

```text
I need a single-file PowerShell script that removes backed-up user data from
one or more Veeam Backup for Microsoft 365 8 repositories, for a list of
users in a CSV file. The customer has moved these users to Veeam Data Cloud
and wants the legacy repository emptied for them. Use the cmdlet contract
block I have pasted alongside this prompt as ground truth for every cmdlet,
parameter and property.

Reference docs:
https://helpcenter.veeam.com/docs/vbo365/powershell/remove-vboentitydata.html
https://helpcenter.veeam.com/docs/vbo365/powershell/get-vboentitydata.html

Target: Windows PowerShell 5.1 and PowerShell 7 on the VB365 server, with
the Veeam.Archiver.PowerShell module. No PowerShell-7-only syntax (no
ternary, no null-coalescing, no ForEach-Object -Parallel).

The overriding design goal: the script must never remove data it was not
certain it was asked to remove. Anything ambiguous is an error for that
user, never a guess.

Requirements:

1. Modes: a -Mode parameter with values Inventory, Preview and Remove.
   Inventory is the default and is read-only. Preview runs
   Remove-VBOEntityData with -WhatIf. Only Remove deletes anything.
2. Input: a CSV with a UserPrincipalName column (-UsersCsv). Trim values,
   reject blank and duplicate rows, and report them before doing anything
   else. Never look users up by display name.
3. Repositories: -RepositoryName (one or more, primary repositories) and
   -LongTermRepositoryName (zero or more, backup copy targets, fetched with
   -LongTerm). Resolve every repository at startup and exit before touching
   any user if one is not found or a name matches more than one repository.
4. Resolving users: resolve each UPN against the backed-up data, not the
   live tenant, because leavers are usually already gone from Entra ID.
   Call Get-VBOEntityData -Type User once per repository, cache the result,
   and match on the email/UPN property with an exact, case-insensitive
   comparison. The property name is UNVERIFIED: on the first object, use
   Get-Member to find it, and if there is no single clear email/UPN
   property, STOP and print the available properties so I can pass the
   right one with -MatchProperty. Zero matches is "not in this repository"
   (report, not an error). More than one match is an error for that user
   in that repository. Never pass an array to -User.
5. Workloads: -Mailbox, -ArchiveMailbox, -OneDrive and -Sites switches,
   passed straight through. If none is given, exit with an error. Do not
   default to "all".
6. Inventory output: for every user and repository, write a row to a
   results CSV with UPN, repository, repository type (primary or long-term),
   match status, the matched object's display name and the workload flags
   if those properties exist. Print a summary: users found, not found,
   ambiguous, per repository.
7. Pre-flight before Preview or Remove: warn that legacy jobs pointing at
   these repositories must be disabled or must no longer include these
   users, or the data will simply be backed up again. Do not try to change
   jobs yourself. If a repository reports immutability (check with
   Get-Member, do not guess the property), skip it with a clear message
   that data must age out through retention. If the property cannot be
   found, say so and carry on, because the cmdlet itself will refuse.
8. Remove safeguards: require -Confirm:$true behaviour by default. Offer
   -Force to suppress the per-user prompt only when the operator has first
   typed the exact number of users to be removed at a Read-Host prompt.
   Support -Limit N to process only the first N resolved users, so a canary
   run on one non-critical user is the obvious first step. Never use
   -RunAsync. Process users one at a time.
9. Resilience: a failure for one user or repository must not stop the
   others. Catch the error, record it in the results CSV, continue. Exit
   with a non-zero code if anything failed or was ambiguous.
10. Verification after Remove: re-run the Get-VBOEntityData lookup for each
   removed user and record whether the data is still present. Also list the
   data management sessions started during the run from
   Get-VBODataManagementSession, with whatever status and result properties
   exist (check with Get-Member).
11. Logging: Start-Transcript to a timestamped file next to the results
   CSV. Write-Host is fine for the summary, but every per-user action and
   error must also be in the results CSV.
12. Help: comment-based help at the top with a synopsis, every parameter,
   and examples in this order: Inventory run, Preview run, canary Remove
   with -Limit 1, full Remove.

Before showing me the result, write Pester 5 tests with every Veeam cmdlet
mocked, covering: user found in one repository; user not found; two entity
objects with the same UPN; the email property missing from the mocked
object; Remove-VBOEntityData throwing for one user while the next succeeds;
-Limit 1; Inventory and Preview making no call to Remove-VBOEntityData
without -WhatIf. Confirm all tests pass. Do not test against a real VB365
server: I will run Inventory and Preview against one myself.

(If your model cannot execute code, delete the paragraph above and run
those cases yourself.)
```

## Notes

* **Run in this order: Inventory, Preview, Remove with `-Limit 1` on a non-critical user, check that user in Veeam Explorer, then the full list.** Removal is irreversible.
* Before removing anything, check that the users and their restore points are present and restorable in Veeam Data Cloud, and that no legal hold or retention requirement still applies to the legacy copy.
* Disable the legacy jobs, or take these users out of their scope, first. Otherwise the next run backs the data up again.
* **Immutable repositories cannot be cleaned up this way.** `Remove-VBOEntityData` refuses them, and immutability cannot be switched off once it is on. The data has to age out through retention.
* **Backup copy repositories are separate.** If the customer has a backup copy job to Azure Archive or another long-term repository, pass it with `-LongTermRepositoryName` or the data and its cost stay there. Azure Cool and Archive tiers also have early-deletion charges.
* **Storage usage may not drop straight away.** Besides Veeam processing time and Azure metrics lag, check whether the storage account has blob soft delete or versioning turned on. If it does, deleted data is still billed until that retention period ends.
* Do not use `-Name "Display Name"`, as many examples online do. Display names are not unique, and a duplicate turns the result into an array that `Remove-VBOEntityData` rejects. The prompt matches on UPN for this reason.
* Do not resolve users with `Get-VBOOrganizationUser` alone. It queries the live tenant, so users already deleted from Entra ID are not found and their data is skipped.
* `-Sites` covers the user's own sites data. SharePoint, Group and Team sites the user worked in are separate entities and are out of scope for this prompt.
* What is not yet confirmed on a live server: the property names on the `Get-VBOEntityData` user object (email/UPN, display name, workload flags), the repository immutability property, and the `Get-VBODataManagementSession` properties. The prompt makes the script discover these with `Get-Member` and stop if they are missing. Check the Inventory output against the VB365 console before you trust it.
* Models tend to get these wrong: looking users up by display name, defaulting to all workloads, adding `-Confirm:$false` everywhere, using `-RunAsync` in a loop, and "helpfully" disabling jobs. The requirements call each of these out, so check for them in the output.

## Disclaimer

See the [read this first](../README.md#%EF%B8%8F-read-this-first) section in the root README. This prompt is a starting point, not a finished solution. The script it produces deletes backup data permanently. Review and test all generated output, and run it in Inventory and Preview mode first, before running it against production.
