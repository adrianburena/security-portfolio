# Operation Dead Deploy: Assessing Azure governance in a training tenant

## Scenario

The Mad Hat Labs scenario describes a junior intern who was given temporary Contributor access to deploy a test environment and left before its governance configuration was reviewed.

I assessed the deployed environment, documented the visible configuration, and identified governance findings and items requiring further validation. The deployment story comes from the lab briefing; the observations below come from the Azure portal.

## Environment

- Platform and tools: Azure portal, resource groups, storage accounts, deployment history, and Azure Policy.
- Environment: live multi-user Azure training tenant.
- Access: Reader; "operative" is the lab's clearance label.
- Scope: read-only assessment. No configuration changes or remediation were performed.

## Investigation

**Stage 1: Locating the test environment**

I reviewed the resource group list and identified `testdeploy123`, which does not follow the lab's naming convention:

`{resource-type}-{workload}-{environment}-{region}-{instance}`

The resource group was located in East US, while most groups visible in the list were in Central US. This difference requires validation against approved-region requirements; the predominant region alone does not establish a violation.

![Resource group testdeploy123 and its East US location](Images/01-operation-dead-deploy/stage1-name-audit.JPG)

**Stage 2: Inspecting the storage account and tags**

The resource group contained one storage account. Its overview showed `owner: intern-jenkins` and `cost-center: unspecified`. The cost-center value does not identify an accountable cost center and should be replaced with a validated value.

The storage account was also located in East US. This is distinct from the resource group's location: a resource group's region identifies where its metadata is stored, and its resources can be deployed in other regions. [Microsoft Learn: Resource group location](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/get-started/how-azure-resource-manager-works).

![Storage account overview showing owner and unspecified cost-center tags](Images/01-operation-dead-deploy/stage2-tags-audit.JPG)

**Stage 3: Reviewing deployment history**

The deployment list showed one successful deployment, last modified on 24 May 2026 at 15:42:45, with a duration of 20 seconds and 413 milliseconds. These values are recorded as displayed in the portal; the screenshot does not establish the time zone.

This record provides a starting point for a deployment timeline. To identify the initiating identity, I would correlate deployment details with Activity Log events and inspect the `caller` field. The captured deployment list alone does not establish who performed the deployment or confirm related security alerts. [Microsoft Learn: Activity Log event schema](https://learn.microsoft.com/en-us/azure/azure-monitor/fundamentals/activity-log-schema).

![Successful deployment with its displayed timestamp and duration](Images/01-operation-dead-deploy/stage3-trace-deployment.JPG)

**Stage 4: Reviewing the naming policy**

The policy assignment named `Naming Convention` had its `effect` parameter set to `Audit`. Audit reports non-compliance without blocking a matching deployment request. For matching create or update requests, it also records a warning event in Activity Log; notifications require separately configured alerting. [Microsoft Learn: Audit effect](https://learn.microsoft.com/en-us/azure/governance/policy/concepts/effect-audit).

This confirms the observed assignment uses an auditing control. It does not establish whether that assignment applied to the resource group when it was created; the policy definition, assignment scope, and historical configuration require validation.

![Naming Convention policy assignment with effect set to Audit](Images/01-operation-dead-deploy/stage4-broken-policies.JPG)

## What broke / what surprised me

A naming policy can be present while still allowing non-compliant deployments. Whether this represents an enforcement gap depends on the governance requirement: Audit can be appropriate during validation, while mandatory naming may require Deny after the definition, scope, and exemptions have been tested.

The unspecified cost-center tag also showed how a small configuration omission can weaken cost attribution. Deployment history added useful timing evidence, but identifying the initiating identity would require additional log correlation.

## Findings and recommendations

| Finding | Evidence | Impact | Recommendation |
| --- | --- | --- | --- |
| Resource group does not follow the lab naming convention | Resource group name: `testdeploy123` | Reduces consistency and makes resource identification harder | Validate the applicable naming rule and plan remediation for the existing group; evaluate enforcement for future deployments |
| Deployment region requires validation | Resource group and storage account both show East US; most visible groups show Central US | Compliance with approved-region requirements remains unconfirmed | Confirm the approved regions for both resource groups and storage accounts before classifying this as a violation |
| Cost center is unspecified | Storage account tag: `cost-center: unspecified` | Prevents reliable attribution to an accountable cost center | Confirm the correct cost center with the owner and validate required tag values, including rejection of placeholders |
| Naming policy audits rather than blocks | `Naming Convention` assignment: `effect = Audit` | This assignment does not block matching non-compliant deployment requests | If preventive enforcement is required, test Deny after validating the definition, assignment scope, enforcement mode, and exemptions |

## What I learned

- Tags are useful only when their values are meaningful. A present but unspecified cost-center tag does not provide reliable cost attribution.
- A region difference is an observation to investigate, not sufficient evidence of a governance violation.
- Audit detects non-compliance; Deny can block matching create or update requests when enforcement is enabled. Alert notifications require their own configuration.
- Deployment timestamps help reconstruct events, but attribution requires supporting activity records.
- A read-only assessment should distinguish observed configuration, the lab's scenario, and conclusions that still require validation.
