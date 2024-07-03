# Let's discuss Entra ID role-assignable groups.

## Introduction

Role-assignable groups in Entra ID (P1 or P2 licensing required) are objects with subtle, but important difference, distinguishing them from other ordinary groups. Namely, during their creation, the ```isRoleAssignable``` attribute is set to ```True```.
It is an immutable attribute, available only during the group creation.<br> Groups created as role-assignable stay as role-assignable for their entire lifecycle. With such attribute populated, these groups become eligible for direct assignments to Entra ID directory roles and more. 

## Primary use case
Think about the scenario where instead of having to assign each user individually to an Entra ID administrative directory role (e.g. 'Security Administrator' and [lots of others](https://learn.microsoft.com/en-us/entra/identity/role-based-access-control/permissions-reference)), you designate a role-assignable group and carefully manage membership of that group.<br>
The resultant effect is the same, user gets the directory role, however, role-assignable groups have some additional security assurances applied by default.

![image](https://github.com/lucas-ko/MSGraphQueries/assets/58331927/b19f8a59-fba5-4164-9180-812ee2b52c7d)

## Security benefits
What are the security assurances when directory roles are managed with role-assignable groups?
- Role-assignable groups can't be managed by directory roles that can modify "ordinary" groups, i.e. ```Groups Administrator```, ```User Administrator```, ```Directory Writers```, ```Identity Governance Administrator```, ```Partner Tier1 Support```, ```Partner Tier2 Support```, ```Exchange Administrator``` (only M365 groups), ```Intune Administrator``` (only security groups) and any custom roles with ```microsoft.directory/groups/members/update``` permission.
- Only ```Global Administrators``` or ```Privileged Role Administrator``` roles can manage membership and ownership of role-assignable groups. This automatically makes such groups manageable only to most critical directory roles (i.e. Tier 0 control plane admins), limits scope of administrative authority over them and reduces the risk of intentional or inadvertent unauthorized group membership modifications (which usually lead to elevation of privilege attacks).
- Similarly, only service principals with ```RoleManagement.ReadWrite.Directory``` Graph API permission will be able to manage role-assignable groups. Service principals with ```Group.ReadWrite.All```, ```GroupMember.ReadWrite.All``` or ```Directory.ReadWrite.All``` permission won't have management capabilities.
- Role-assignable groups can't be dynamic. By blocking this capability, unintended updates of group membership based on dynamic membership rules, with security dependencies on source object attributes, are out of the picture!
- Permanent (not eligible, if you use Entra ID PIM and PIM for Groups!) members of role-assignable groups become protected too! Scope of administrative authority over these accounts becomes smaller. For example, password and MFA resets can be performed by ```Privileged Authentication Administrator``` and ```Global Administrator``` roles only. In normal group setup, several more roles have access to [perform these actions](https://learn.microsoft.com/en-us/entra/identity/role-based-access-control/privileged-roles-permissions?tabs=admin-center#who-can-reset-passwords).
  
## Alternative use case(s)
What if you create a group with ```isRoleAssignable``` attribute enabled and decide to not assign them to any Entra ID directory role?<br>
Are these groups usable elsewhere? Yes!

Let's assume you have set of critical, baseline conditional access policies (e.g. require multi-factor authentication with specific strength for all users) in which you have to make some exclusions.<br>
How do you usually do it? <br>You probably create ordinary (not role-assignable) security group and add some members. It is a common approach, with some residual risks stemming from the use of "standard" security or M365 group.<br>
**Residual risk in this context is indirect delegation of control over these critical conditional access policy exclusions. Such indirect control is given to all Entra ID directory roles that are allowed to modify groups' membership.<br> To be more precise, 'tier 1 - management plane' roles have control over identity configuration elements that shall be exclusively reserved to 'tier 0 - control plane' ones.<br>
Put it differently, you implicitly delegate scope of application of your critical conditional access policies!**

Using role-assignable group to control the CA policy exclusions would be more optimal, risk-wise. With such configuration, only two aforementioned critical directory roles (```Global Administrators```, ```Privileged Role Administrator```) would be in direct control of group membership and exclusion scope of your critical CA policies.

![image](https://github.com/lucas-ko/MSGraphQueries/assets/58331927/c09cb278-2f05-43f5-81a7-5d5ebff98e1f)

Role-assignable groups can be used every time you worry about management scope of group membership or ownership, and you want to tighten it down. Scenarios like Microsoft Defender XDR unified RBAC role assignments, third party line of business application roles and many others.

## Things to watch out for

There is tenant-wide limit of 500 role-assignable groups. Keep that in mind and use them only if you really require additional assurances related to management of group membership/ownership.

---
All work is licensed under a [Creative Commons Attribution 4.0 International License][cc-by].

[![CC BY 4.0][cc-by-image]][cc-by]

[cc-by]: http://creativecommons.org/licenses/by/4.0/
[cc-by-image]: https://i.creativecommons.org/l/by/4.0/88x31.png
[cc-by-shield]: https://img.shields.io/badge/License-CC%20BY%204.0-lightgrey.svg
