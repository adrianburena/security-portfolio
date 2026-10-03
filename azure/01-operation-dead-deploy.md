# The Operation Dead Deploy: Investigating a four-stage forensic operation in Azure

## Scenario

Two days ago, a junior intern at Mad Hat Labs was given temporary contributor access to spin up a "test environment" for an experiment that would not be presented to leadership. The intern was not familiar with the company's governance standards and cut several corners. They deployed quickly and left for the weekend.
The scope is to assess the test environment, determine whether governance is properly defined, and document any findings.

## Environment
Azure, resource groups, storage, access level: operative.
Environment: live multi-user Azure training tenant, Reader access.

## Investigation
**Stage 1**: Locating the test environment
In the first stage, the deployed resource is sought. For that, the Resource groups option in the left blade is selected. By browsing through all the group resources, a resource group named `testdeploy123` is found that does not follow the naming convention:
{resource-type}-{workload}-{environment}-{region}-{instance}.

As a second finding, the resource group is located in East US while most of the other resource groups are in Central US. The correct location should be evaluated against approved-region requirements. The resource group location refers to its metadata; the storage account location is checked separately.

![Wrong Naming Convention](Images/01-operation-dead-deploy/stage1-name-audit.JPG)

**Stage 2**: The payload is inspected.
After inspecting the resource group, there is one storage account linked to it. Also, there is one successful deployment log. Inspecting the storage, the tags are analyzed, revealing ownership, cost center, and lifecycle. The visible tags are `owner: intern-jenkins` and `cost-center: unspecified`, so the cost center has not been properly assigned. The storage account is also located in East US.

![Tags](Images/01-operation-dead-deploy/stage2-tags-audit.JPG)

**Stage 3**: Tracing the deployment.
On the deployment blade, the deployment record shows timestamps to correlate with security alerts and create an incident timeline. It can also reveal who is responsible for each deployment.

![Deployment timestamps](Images/01-operation-dead-deploy/stage3-trace-deployment.JPG)

**Stage 4**: Policies.
Navigating to the resource group's "Policies" blade, there is an existing policy about naming convention. As it was found, the parameter value was set to "Audit" instead of "Deny." This effect records non-compliance without blocking the deployment. For matching create or update requests, Audit also records a warning event in Activity Log; alert notifications require separate configuration.

![Policies](Images/01-operation-dead-deploy/stage4-broken-policies.JPG)

## What broke / what surprised me
The importance of properly configured policies is vital to avoid undesired deployments and other issues. Also, proper tagging can help personnel understand a resource's lifecycle, cost center, and ownership in a timely manner. As for the deployment blade, we can get a full picture of the story behind the deployment. If naming compliance is intended to be mandatory, the policy should be evaluated for enforcement with the Deny effect after validating its scope and any required exemptions.

## Findings and recommendations
|   Finding |   Evidence    |   Impact  |   Recommendation  |
| --- | --- | --- | --- |
| Naming convention violation   | Resource group name: `testdeploy123`   |   Reduced consistency and resource identification | Enforce naming policy |
|  Region may differ from expected deployment region    | Resource group and storage account: East US; most visible groups: Central US    | Requires validation against governance requirements   | Confirm approved deployment region    |
|   Cost-center tag improperly assigned | Storage account tag: `cost-center: unspecified` | Weakens ownership/cost attribution    | Correct mandatory tagging |
|   Naming policy uses Audit    |   `Naming Convention` policy assignment: `effect = Audit`    |   Non-compliant resources can still be deployed   | Evaluate Deny enforcement |

## What I learned
- Tags do matter. They help us understand all the resources.
- Proper naming is there for a reason. It's easier to identify any resource and its importance if it is properly named. Real-world scenarios: not following the naming convention when there are many resources might reduce efficiency when searching for specific resources, potential unwanted or unauthorized resources may sit there for days or weeks before someone spots them, and resources may accumulate and cause alert fatigue.
- A policy set to Audit records non-compliance but does not block the deployment, and this could be potentially dangerous if preventive enforcement is required. Real-world scenarios: a policy is in place to allow only basic VMs for testers (B-series), but it has been set to Audit. A developer deploys an N-series VM for testing. This could increase billing because the deployment would go through and record an audit event; an alert notification would require separate configuration. A Deny policy with enforcement enabled would block the non-compliant VM creation request.

End report
