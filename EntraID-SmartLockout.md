## Entra ID - smart lockout - protects your users from malicious account lockouts!
I recently had several conversations related to smart lockout feature in Entra ID.  Based on those, it occurred to me that inner workings of this feature are not as widely known as I assumed.<br>

## **TL;DR<br>**
Smart lockout is a capability of Entra ID that makes a given user account appear locked out for certain entities, while allowing legitimate users to successfully authenticate.<br>
By slowing down an attacker, it raises the cost of successful brute force attack on primary authentication factor (it's unfortunately still a password in majority of the cases).

## How does Entra ID realize this?

The secret lies in clever management of account metadata responsible for lockout: 
- each Entra ID user account has two separate lockout counters;
- each lockout counter is assigned to a specific location - either "unknown" or "familiar" one;
- "familiar" location keeps track of IP address ranges from which user has successfully authenticated in the past<sup>*</sup>, everything else is "unknown".

<sup>* \- the period of observation is not disclosed but it is fair to assume 30-90 days</sup>

```mermaid
flowchart LR
    Start@{ shape: circle, label: "Authentication attempt" }
    Is_IPFamiliar@{ shape: diamond, label: "Is source IP in familiar location?" }
    Fam_Is_AccountLocked@{ shape: diamond, label: "Is account locked for auth. attempts from familiar locations?" }
    Unfam_Is_AccountLocked@{ shape: diamond, label: "Is account locked for auth. attempts from unknown locations?" }
    Fam_Deny@{ shape: rounded, label: "Deny authentication" }
    Unfam_Deny@{ shape: rounded, label: "Deny authentication" }
    Unfam_CredsOK@{ shape: diamond, label: "Are credentials valid?" }
    Fam_CredsOK@{ shape: diamond, label: "Are credentials valid?" }
    Fam_AllowAuth@{ shape: rounded, label: "Accept authentication" }
    Unfam_AllowAuth@{ shape: rounded, label: "Accept authentication" }
    Fam_AddIP@{ shape: rounded, label: "Add source IP to familiar locations" }
    Unfam_AddIP@{ shape: rounded, label: "Add source IP to familiar locations" }
    Fam_IncrementCounter@{ shape: rounded, label: "Increment bad auth. attempt counter for familiar locations" }
    Unfam_IncrementCounter@{ shape: rounded, label: "Increment bad auth. attempt counter for unknown locations" }
    Unfam_Is_Suspicious@{ shape: diamond, label: "Is activity suspicious - matches known attack patterns?" }
Start --> Is_IPFamiliar
Is_IPFamiliar-->|Yes|Fam_Is_AccountLocked
Is_IPFamiliar-->|No|Unfam_Is_AccountLocked
Fam_Is_AccountLocked-->|Yes|Fam_Deny
Fam_Is_AccountLocked-->|No|Fam_CredsOK
Fam_CredsOK-->|No|Fam_Deny
Fam_CredsOK-->|Yes|Fam_AllowAuth
Fam_AllowAuth-->Fam_AddIP
Fam_Deny-->Fam_IncrementCounter
Unfam_Is_AccountLocked-->|Yes|Unfam_Deny
Unfam_Is_AccountLocked-->|No|Unfam_Is_Suspicious
Unfam_Is_Suspicious-->|Yes|Unfam_Deny
Unfam_Is_Suspicious-->|No|Unfam_CredsOK
Unfam_CredsOK-->|No|Unfam_Deny
Unfam_CredsOK-->|Yes|Unfam_AllowAuth
Unfam_AllowAuth-->Unfam_AddIP
Unfam_Deny-->Unfam_IncrementCounter
```

Smart lockout is enabled for every Entra ID customer but it's not configurable for free tenants.
For tenants licensed with Premium P1 or P2, some configuration capabilities become available:

**Lockout threshold** - Maximum number of bad authentication attempts over which the account is locked out. If the first sign-in after a lockout also fails, the account locks out again. 

**Lockout duration** - Minimum lockout duration in seconds (during initial lockout). Subsequent lockouts are increasingly longer.

![image](https://github.com/user-attachments/assets/dcea6684-228c-4fed-824d-1ba727997ba0)

You might say, _"That's cool, but what actually happens if I am hybrid and use password hash synchronization (PHS), will my affected users' on-premises accounts also be locked out?"_.

Great question! Let's consider two scenarios:

**Scenario 1:<br>**
- Attacker tries to brute force user password from location that is unknown for the user, lockout counter exceeds the limit.

**Result:<br>**
- Attacker is locked out. User authenticating from familiar location can authenticate successfully to all services relying on Entra ID, as well as to on-premises workloads.

**Scenario 2:<br>**
- Attacker attempts to brute force user password from location that is familiar for the user and lockout counter exceeds the limit.

**Result:<br>**
- Both attacker and legitimate user are locked out. During the lockout, user cannot successfully authenticate to all services relying on Entra ID (e.g., Microsoft 365, Azure Resource Manager, 3rd party integrated SaaS apps). 
Because affected user's lockout status is not replicated to on-premises Active Directory, they can still successfully authenticate to all on-prem workloads relying on Active Directory.

**What about passwordless users? Are their account susceptible to brute force attempts and malicious lockouts?**<br>
Unfortunately yes, as currently you can't create an user object without specyfing its password, below creation attempt via _New-MGBetaUser_ proves that.<br>

![image](https://github.com/user-attachments/assets/644a33b7-a658-4729-9596-58a602d71b43)

>[!NOTE]
>If you are using pass-through-authentication or ADFS (you really need that PHS project going!), above scenarios become a bit more complex as they involve real-time password verification on-premises. To avoid unnecessary account lockout, you should set lockout threshold in Entra ID to be lower than for on-premises domain.

>[!TIP]
>Microsoft Entra ID also protects against attacks by analyzing more signals during each authentication attempt. Assessed data includes source IP reputation and associated malicious activity.
>If the sign-in is coming from an suspicious IP, regardless if credentials are correct, Entra returns [AADSTS50053](https://learn.microsoft.com/en-us/entra/identity-platform/reference-error-codes#aadsts-error-codes) error code.
