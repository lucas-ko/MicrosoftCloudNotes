## Entra ID - smart lockout - protect your users from malicious account lockouts!
I recently had several conversations related to smart lockout feature in Entra ID.  Based on those, it occurred to me that inner workings of this feature are not as widely know as I assumed.<br>

**TL;DR -  Smart lockout is a capability of Entra ID that makes a given user account appear locked out for certain entities, while allowing legitimate users to successfully authenticate.**

## How does Entra ID realize this?

The secret lies in clever management of account metadata responsible for lockout: 
- each Entra ID user account has two separate lockout counters;
- each lockout counter is assigned to a specific location - either "unfamiliar" or "familiar" one;
- "familiar" location keeps track of IP address ranges from which user has successfully authenticated in the past<sup>*</sup>, everything else is "unfamiliar".

<sup>* \- the period of observation is not disclosed but it is fair to assume 30-90 days</sup>

Smart lockout is enabled for every Entra ID customer but it's not configurable for free tenants.
For tenants licensed with Premium P1 or P2, some configuration capabilities become available:
Lockout threshold - Maximum number of bad authentication attempts over which the account is locked out. If the first sign-in after a lockout also fails, the account locks out again. 
Lockout duration - Minimum lockout duration in seconds (during initial lockout). Subsequent lockouts are increasingly longer.

![image](https://github.com/user-attachments/assets/dcea6684-228c-4fed-824d-1ba727997ba0)

You might say, _"That's cool, but what actually happens if I am hybrid and use password hash synchronization (PHS), will my affected users' on-premises accounts also be locked out?"_.

Great question! Let's consider two scenarios:

**Scenario 1:<br>**
- Attacker tries to brute force user password from location that is unfamiliar for the user, lockout counter exceeds the limit.

**Result:<br>**
- Attacker is locked out. User authenticating from familiar location can authenticate successfully to all services relying on Entra ID, as well as to on-premises workloads.

**Scenario 2:<br>**
- Attacker attempts to brute force user password from location that is familiar for the user and lockout counter exceeds the limit.

**Result:<br>**
- Both attacker and legitimate user are locked out. During the lockout, user cannot successfully authenticate to all services relying on Entra ID (e.g., Microsoft 365, Azure Resource Manager, 3rd party integrated SaaS apps). 
Because affected user's lockout status is not replicated to on-premises Active Directory, they can still successfully authenticate to all on-prem workloads relying on AD.

>[!NOTE]
>If you are using pass-through-authentication or ADFS (you really need that PHS project going!), above scenarios become a bit more complex as they involve real-time password verification on-premises. To avoid unnecessary account lockout, you should set lockout threshold in Entra ID to be lower that on-premises domain.

>[!TIP]
>Microsoft Entra ID also protects against attacks by analyzing more signals during each authentication attempt. Assessed data includes source IP reputation and associated malicious activity.
>If the sign-in is coming from an suspicious IP, regardless if credentials are correct, Entra returns [AADSTS50053](https://learn.microsoft.com/en-us/entra/identity-platform/reference-error-codes#aadsts-error-codes) error code. 
