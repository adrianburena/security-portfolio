# The Operation Dead Deploy:Investigating a four-stage forensic operation in Azure

## Scenario

Two days ago, a junior intern at Mad Hat Labs was given temporary contributor access to spin up a "test environment" for an experiment that would not be presented to leadership. The intern was not familiar with the company's governance standards and they cut several corners. They deployed quickly and left for the weekend.
The scope is to asses the test enviroment, identify the governance is properly defined and document any findings.

## Environment
Azure, Resource groups, storage, Access Level: Operative. 
Enviroment: "live multi-user Azure training tenant, Reader access."

## Investigation
**Stage 1**:Locating the test enviroment
On a fist stage, the deployed resouce is sought. For that, Resource groups in the left blade is selected. Surfing through all the group resources, resource is found not following the naming convention:
 {resource-type}-{workload}-{environment}-{region}-{instance}.

As a second finding, it's been set up in EastUS Location while most of the other resources are on CentralUS. Correct location to be evaluated.

![Wrong Naming Convention](Images/01-operation-dead-deploy/stage1-name-audit.JPG)

**Stage 2**: The payload is inspected.
After inspecting the resource group, there's one storage account linked to it. Also, there's 1 succesful deployment log. Inspecting the storage in there, tags are analyzed revealing the ownership, cost center and lifycle. Cost-center hasn't been properly assinged.

![Tags](Images\01-operation-dead-deploy\stage2-tags-audit.JPG)

**Stage 3**: Tracing the deployment.
On the deployment blade, the deployment record shows timestaps to correlate with security alerts and create an incident timeline. It can also reveal the responsible for each deployment.

![Deployment timestamps](Images\01-operation-dead-deploy\stage3-trace-deployment.JPG)

**Stage 4**: Policies.
Navigating to the resource group's "Policies" blade, there's existing policy about naming convention. As it was found, the parameter value was set to "Audit" instead of "Deny". The effects for this is an alert instead of a denied deployment.

![Deployment timestamps](Images\01-operation-dead-deploy\stage4-broken-policies.JPG)

## What broke / what surprised me
The importance of properly configured policies is vital to avoid undesired deployments and other issues. Also, proper tagging can help personel to understand a resource lifecycle, cost center and ownership in a speedy manner and as for the deployment blade, we can get a full picture of the story behind the deployment.

## Findings and recommendations
Policy needs to be fixed and set to "Deny" instead of "Audit" to avoid wrong naming convention in the future.
Setting a proper costcenter/location is essential before any deployment.

## What I learned
- Tags do matter. Tags help us understand all the resources
- Proper naming is there for a reason. Its easier to identify any resource and its importance if its properly named.
- A policy set to Audit will fire the alarms but won't stop someting from happening and this could be potentialy dangerous. 
