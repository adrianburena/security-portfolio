# Privilege Audit - RBAC

## Scenario
After a previous incident involving stolen identity credentials, a confused deputy attack, a malicious API deployment, and a suspicious URI, the management board requested a privilege audit for resource-based access. This report covers Azure RBAC.

## Environment
**Azure Cloud**

Services/tools: Microsoft Entra ID, app registrations, enterprise applications, and API permissions.

Environment: live multi-user Azure training tenant.

Status: Field Audit in Progress

Clearance: Operative (Reader access)

## Investigation
The audit path:

Access Control -> Command Line -> Resource Graph -> PIM -> Findings

Performed a role-based access control audit across a live Azure tenant using four methods: 

a) Portal role assignment export <br>
The IAM blade is part of every scope. For this first strategy, the role assignments at a subscription level are exported to a .csv file and downloaded.

One account with many redundant Owner roles across multiple resources was found (RBAC-01). These roles may have been assigned by a recurring script, manually, or through dynamic groups.

![Portal Blade](azure/Images/03-privilege-audit/01-portal-blade.JPG)

b) Azure CLI enumeration <br>
The command line interface (CLI) can use commands like Get-AzRoleAssignment or az role assignment list --resource-group ResourceGroupName to enumerate the same assignments as the IAM blade -> export .csv
The advantage offered by the CLI is that it is scriptable and it is repeatable.

![CLI](azure/Images/03-privilege-audit/02-cli.JPG)

A .JSON file can be exported for full analysis. As a second finding (RBAC-02), there is an account with no principalName. This is an orphaned account that was deleted and whose permissions were not revoked.

![Orphaned Account](azure/Images/03-privilege-audit/03-orphaned-account.JPG)

c) Resource Graph KQL sweep <br>
The CLI and IAM blade both work with only one scope at a time. Resource Graph offers the possibility to audit the whole tenant in a single query with KQL (Kusto Query Language).

authorizationresources

| where type =~ 'microsoft.authorization/roleassignments'

| extend principalId = tostring(properties.principalId)

| extend description = properties.description

| where principalId == '<insert orphaned principal ID>'

| project name, principalId, principalType = properties.principalType, scope = properties.scope, description

In this case, a specific account was searched for during the audit.

![KQL](azure/Images/03-privilege-audit/04-kql.JPG)

d) PIM eligible-versus-active export. 

All of the previous methods were focused on active access. For eligible access, there are a few methods that could be used. <br>
CLI: <br>
Get-AzRoleEligibilitySchedule (To list eligible assignments) <br>
Get-AzRoleAssignmentSchedule (To list active role assignments) <br>

PIM: <br>
This is the route followed: <br>
PIM -> Manage -> Subscription -> Resource Group -> Manage -> Assignments -> Export.

A deleted account with unrevoked permanent Reader access was found (RBAC-02).

![PIM](azure/Images/03-privilege-audit/05-pim.JPG)

The Hunt

Until now, the findings have occurred with standard Reader access. The account is eligible for elevated privileges on a hidden resource group, so a request was filed in order to inspect the group.

![The Hunt - Resouce Group](azure/Images/03-privilege-audit/06-resource-group.JPG)

The file for the hidden resource group is exported and analyzed.

![The Hunt - Export](azure/Images/03-privilege-audit/07-export.JPG)

An account was found with permanent Owner privileges (RBAC-03).

![Owner found](azure/Images/03-privilege-audit/08-owner.JPG)

It can also be found using the CLI.

![The Hunt - CLI](azure/Images/03-privilege-audit/09-cli-rbac-03.JPG)

And using Resource Graph with KQL, the same result.

![The Hunt - Resouce Graph](azure/Images/03-privilege-audit/10-resource-graph-verification.JPG)

## What broke / what surprised me
There are many different ways to retrieve information in Azure. Sometimes, for "convenience," orphaned accounts are not displayed with some methods. Also, the different methods offer different advantages over others.

## Findings and recommendations
| ID | Severity | Score | Finding | Evidence | Impact | Recommendation |
| --- | --- | ---: | --- | --- | --- | --- |
| RBAC-01 | HIGH | 15/25 | Excessive and redundant privileged access | Active identity holds multiple redundant Owner role assignments across several Azure resources and scopes. | Excessive privileged access increases the blast radius of account compromise or misuse. A compromised identity could modify resources or access controls across multiple affected scopes. | Remove redundant Owner assignments and replace them with the narrowest job-function role at the narrowest required scope. Prefer group-based RBAC assignments over repeated direct user assignments. |
| RBAC-03 | HIGH | 15/25 | Standing permanent Owner access | Active identity has permanent Owner privileges on a resource group instead of PIM-eligible access. | Highly privileged permissions remain continuously available. Account compromise or misuse could result in resource modification, deletion, or unauthorized access delegation. | Move standing Owner access to PIM-eligible access, time-limited activation and, approval if necessary. |
| RBAC-02 | LOW | 4/25 | Orphaned RBAC role assignment with permanent reader access | An RBAC assignment references a principal that has been deleted and can no longer be resolved to a principal name. | The stale assignment indicates incomplete identity deprovisioning and complicates access reviews, auditing, and verification of effective permissions. | Verify that the principal has been deleted and revoke the orphaned assignment. Include orphaned-assignment detection in recurring access reviews. |

## What I learned
This are the advantages/disadvantages of each method:

| Method               | Sees                                        | Misses                              |
|----------------------|---------------------------------------------|-------------------------------------|
| IAM blade / export   | Active assignments at scope, inherited      | Group members, orphaned principals  |
| Azure CLI            | Same, plus null principalName (orphans)     | One scope per run                   |
| Resource Graph (KQL) | Whole tenant in one query                   | Eligible assignments                |
| PIM export           | Eligible vs active, activation history      | Assignments outside PIM             |
