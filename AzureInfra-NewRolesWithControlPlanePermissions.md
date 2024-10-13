## Intro

Some interesting changes happened recently (within last year :wink:) in the area of highly privileged (i.e. control plane or Tier 0 in legacy terms) Azure infrastructure RBAC model roles.<br>
Up until now, the classic trio of control plane privileged roles with direct control over Azure infrastructure was limited to:

- **_Owner_**
- **_User Access Administrator_**
- Entra ID **_Global Administrator_** (via [single-click elevation](https://learn.microsoft.com/en-us/azure/role-based-access-control/elevate-access-global-admin)❕)

>[!TIP]
>Indirect elevation paths to control plane starting with less powerful roles are essentially unlimited, but that's a topic for different post.<br>
>To get a glimpse, see these excellent articles by Andy Robbins and Karl Fosaaen :
>- [https://medium.com/specter-ops-posts/automating-azure-abuse-research-part-1-30b0eca33418](https://medium.com/specter-ops-posts/automating-azure-abuse-research-part-1-30b0eca33418)
>- [https://www.netspi.com/blog/technical-blog/cloud-pentesting/azure-logic-app-contributor-escalation-to-root-owner](https://www.netspi.com/blog/technical-blog/cloud-pentesting/azure-automation-account-connections/)

Since then, two new roles have been added:

- [Role Based Access Control Administrator](https://learn.microsoft.com/en-us/azure/role-based-access-control/built-in-roles/privileged#role-based-access-control-administrator)
- [Reservations Administrator](https://learn.microsoft.com/en-us/azure/role-based-access-control/built-in-roles/privileged#reservations-administrator)

## Reservations Administrator

Let's dissect **_Reservations Administrator_** first.<br>
Is it really a control plane role? To answer this, we need to look at its definition:

```
{
  "assignableScopes": [
    "/providers/Microsoft.Capacity"
  ],
<...redacted for brevity...>
  "permissions": [
    {
      "actions": [
        "Microsoft.Capacity/*/read",
        "Microsoft.Capacity/*/action",
        "Microsoft.Capacity/*/write",
        "Microsoft.Authorization/roleAssignments/read",
        "Microsoft.Authorization/roleDefinitions/read",
        "Microsoft.Authorization/roleAssignments/write",
        "Microsoft.Authorization/roleAssignments/delete"
      ],
      "notActions": [],
      "dataActions": [],
      "notDataActions": []
    }
  ],
<...redacted for brevity...>
}
```
All right, it has _<ins>Microsoft.Authorization/roleAssignments/write</ins>_ control plane privileged action allowing assignments of any Azure RBAC role, however, to obtain the full context, one would also need to pay attention to the _<ins>assignableScopes<ins>_ property - set to _<ins>/providers/Microsoft.Capacity</ins>_.<br>That assignable scope value is odd in the context of built-in role, isn't it?<br> However, it makes sense after reviewing the documentation on [Azure Reservations](https://learn.microsoft.com/en-us/azure/cost-management-billing/reservations/view-reservations#who-can-manage-a-reservation-by-default).
![372595407-730fd8c9-3159-430b-ba7d-748772903439](https://github.com/user-attachments/assets/984de9e9-a428-42b2-8614-386d4e98a6aa)<br>

To summarize, **_Reservations Administrator_**, even though it can mange role assignment, it can do so only on Azure reservation objects. Technically it's still a control plane role, but greatly constrained - can't be assigned to resources and constructs other than reservation objects.

>[!TIP]
>To assign **_Reservations Administrator_** role, activate **_User Access Administrator_** (or better **_Role Based Access Control Administrator_**)  and use following Azure CLI command: ```az role assignment create --assignee "<assignee (user or group) ObjectID>" --scope "/providers/Microsoft.Capacity" --role "a8889054-8d42-49c9-bc1c-52486c10e7cd"```.<br>
>Reservations can then be managed in [Cost Management blade in Azure Portal](https://portal.azure.com/#view/Microsoft_Azure_CostManagement/Menu/~/reservations).

## Role Based Access Control Administrator
Onto the second one.<br>
**_Role Based Access Control Administrator_** is, in my opinion, very useful one.<br>
Let's examine role definition first:
```
{
  "assignableScopes": [
    "/"
  ],
<...redacted for brevity...>
  "permissions": [
    {
      "actions": [
        "Microsoft.Authorization/roleAssignments/write",
        "Microsoft.Authorization/roleAssignments/delete",
        "*/read",
        "Microsoft.Support/*"
      ],
      "notActions": [],
      "dataActions": [],
      "notDataActions": []
    }
  ],
<...redacted for brevity...>
}
```
By default, role definition of **Role Based Access Control Administrator** includes following control plane actions:
- ```"Microsoft.Authorization/roleAssignments/write"``` - ability to assign any Azure RBAC role (inc. control plane roles like **_Owner_** or **_User Access Administrator_**) at any scope
- ```"Microsoft.Authorization/roleAssignments/delete"``` - ability to delete assignments of any Azure RBAC role

Above actions make the role highly privileged and member of control plane. That's the default configuration.<br> 
However, the main value of this role comes from the capability to implement Azure RBAC role assignment constrained delegation scenario.

How does it work? Let me describe an imaginary scenario as an example.

>Imagine Anna, who is tasked with administering and operating IAM for her company's Azure infrastructure resources.<br>
>Currently, every time she needs to make a change in role assignments, she has to logon to privileged access workstation, use Privileged Identity Management to activate **_User Access Administrator_** role at a correct scope, and perform the actual role assignment.<br>
>It's only first week of the month and Anna already had to perform over 20 unique role assignments.<br> 
>Tim, who is responsible for setting up the permissions for his team's DevOps pipeline, requests **_User Access Administrator_** to be assigned at the DevOps team subscription scope.<br>
>Anna knows the core tenets of zero trust (**least privilege**, **assume breach** and **verify explicitly**) and decides the request must be implemented in smarter fashion.<br>
>She decides to assign **_Role Based Access Control Administrator_** role to Tim at his team subscription scope.<br>
>Anna also decides the role will be constrained - Tim will be able to assign just the roles delegated to him and assign them only to service principals.<br>

Anna performs following actions in the Azure Portal:<br>

  - She selects <b>'Allow user to only assign selected roles to selected principals'</b> option
  ![376052465-96e862f6-5104-49c9-91f7-01892455d9a1](https://github.com/user-attachments/assets/f39f8706-92aa-444d-9c51-7d1b873462cf)


  - She creates an assignment condition constraining Tim to assign only <b>Contributor</b> role to service principal objects
  ![376053455-f74482ec-831e-4165-9ee0-720208164e80](https://github.com/user-attachments/assets/5737327b-0165-4480-b5ba-f3d391c3f5a5)


By performing above activities Anna accomplished few goals:

- Enabled Tim to manage role assignments within applied guardrails. Tim can be more efficient now, instead of raising requests for role assignments and be blocked until their completion, he can now spend more time on deliverables that can move his team and company goals forward.
- Anna can now spend less time on assigning roles and similarly to Tim, can focus on more important goals.
- Anna did all of this in alignment with security best practices and principles. 

>[!NOTE]
>In which cases does **_Role Based Access Control_** fall shor when compared to **_User Access Administrator_**?
>
>- It can't (yet!) manage eligible role assignments in Privileged Identity Management
>- It can't create new, custom role assignments

## Outro

To summarize, always strive to be aligned with one of the basic Zero Trust tenets - **least privilege**. Minimize the use of **_User Access Administrator_** or **_Owner_** roles and smartly delegate assignment capabilities with **_Role Based Access Control Administrator_**.

Don't be that person! :smiley: <br>
![375982100-2b7b6d7c-a2cd-4b7b-9ced-7a373bd280fe](https://github.com/user-attachments/assets/7a1ab07c-2f80-4ac4-bc10-f8833b5f2d7b)

## Useful resources:

- [Enterprise Access Model](https://learn.microsoft.com/en-us/security/privileged-access-workstations/privileged-access-access-model)
- [Understand role definition structure](https://learn.microsoft.com/en-us/azure/role-based-access-control/role-definitions)
- [Azure infrastructure built-in roles](https://learn.microsoft.com/en-us/azure/role-based-access-control/built-in-roles)
- [Automating Azure abuse research](https://medium.com/specter-ops-posts/automating-azure-abuse-research-part-1-30b0eca33418)
- [Azure Automation Account connections](https://www.netspi.com/blog/technical-blog/cloud-pentesting/azure-automation-account-connections/)

-------------------------------------------------------------------------------------------
All work is licensed under a [Creative Commons Attribution 4.0 International License][cc-by].

[![CC BY 4.0][cc-by-image]][cc-by]

[cc-by]: http://creativecommons.org/licenses/by/4.0/
[cc-by-image]: https://i.creativecommons.org/l/by/4.0/88x31.png
[cc-by-shield]: https://img.shields.io/badge/License-CC%20BY%204.0-lightgrey.svg
