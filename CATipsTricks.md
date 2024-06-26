# Entra ID Conditional Access - tips and tricks
## Intro
Conditional Access policies are at the heart of Entra ID zero trust policy engine enforcement. They collect various signals (identity, device, network location, protocol, client app type, real time sign-in risk, accessed resource) and enable enforcement of company's resource access policies.

![image](https://learn.microsoft.com/en-us/entra/identity/conditional-access/media/plan-conditional-access/conditional-access-overview-how-it-works.png)[^1]
[^1]: Source and credit  - [Microsoft](https://learn.microsoft.com/en-us/entra/identity/conditional-access/plan-conditional-access)

In the article below, I assume the reader has moderate level of familiarity with conditional access concepts.

## Tip #1 - Policies are only processed after successful primary authentication
**Since Conditional Access policies operate on established signals, like user identity, device identity, network location, sign-in session risk and others, they're applied only after successful primary authentication.** <br>
It's important to remember this especially if your access requests are getting blocked by conditional access policy and you aren't sure why. <br>
Don't start with troubleshooting the authentication credentials of the affected users - if they were blocked by conditional access policy, their primary authentication factor (most likely password, sadly) was successfully validated.<br>

Instead, dive into Entra ID sign-in logs and analyze generated errors:
![342750037-10c52ae1-f283-44ec-b56c-2fcf051cccad](https://github.com/lucas-ko/MicrosoftCloudNotes/assets/58331927/55b89697-fe33-47e3-9b10-df1606832234)

## Tip #2 - Legacy authentication blocking - is single line of defense enough?
How does the conditional access policy blocking legacy authentication work?<br>
In short, it blocks protocols allowing client applications (like Outlook 2010 on Windows or built-in Mail app on old version of iOS) to collect user's logon credentials and pass them to a backend service to authenticate the user.<br>
It means every time you see the app prompting you for explicit credentials, there's a high chance that it uses legacy authentication protocol (e.g. SMTP, IMAP4, POP3 and many others). 

Client applications using modern authentication protocols behave differently:
- They send you through their configured, trusted identity provider, which validates your credentials and returns ID or access token that is later consumed by the app/service you're accessing.
- They never collect and process your credentials directly.<br>

Back to conditional access and legacy authentication blocking - you should implement it and strive for minimal exceptions. Good documentation on how to achieve this is available [here](https://learn.microsoft.com/en-us/entra/identity/conditional-access/howto-conditional-access-policy-block-legacy). This solves blocking on authentication platform side.<br>
![342899546-b90298e4-433e-4d9f-af0c-968b130417b0](https://github.com/lucas-ko/MicrosoftCloudNotes/assets/58331927/0bec6de4-6ff1-4a38-8354-dd368f4c99fb)

Shall you do more than this though? Absolutely! Remember, the behavior of conditional access policies is - they apply after successful authentication (see Tip #1).<br>
**In a legacy protocol authentication scenario including attackers who are using password spray or stuffing techniques, if they're blocked by conditional access, it means they were able to successfully validate the password because policy enforcement is performed post-authentication...<br>From IT security prevention and risk reduction perspective, it is better to block such attempts at the protocol level and simply deny credential validation attempts against Entra ID.<br>**

Most of the legacy authentication protocols were disabled by Microsoft in 2022 (see [this](https://techcommunity.microsoft.com/t5/exchange-team-blog/basic-authentication-deprecation-in-exchange-online-time-s-up/ba-p/3695312])), but it's always a good idea to double check your configuration and reinforce its desired state.

To reinforce legacy authentication blocking on service/resource-side (i.e. on Exchange Online or SharePoint Online)
| M365 workload | How to disable legacy auth (PowerShell) | Privilege needed to execute the commands| Relevant documentation|
|--|--|--|--|
| Sharepoint Online and OneDrive |  ```Import-Module Microsoft.Online.SharePoint.PowerShell```<br><br>```Connect-SPOService -URL <YOUR_TENANT_SPO_ADMIN_URL>```<br><br>```#Disable legacy authentication in Sharepoint Online/OneDrive```<br>```Set-SPOTenant -LegacyAuthProtocolsEnabled $False```   | [Sharepoint Administrator](https://learn.microsoft.com/en-us/entra/identity/role-based-access-control/permissions-reference#sharepoint-administrator) Entra ID role  | [MS Learn](https://learn.microsoft.com/en-us/powershell/module/sharepoint-online/set-spotenant?view=sharepoint-ps#-legacyauthprotocolsenabled) |
| Exchange Online |  ```Import-Module ExchangeOnlineManagement```<br><br>```Connect-ExchangeOnline```<br><br>```#Enable modern authentication in ExO organization```<br>```Set-OrganizationConfig -OAuth2ClientProfileEnabled:$True```<br><br>```#Create policy disabling legacy authentication for all available protocols```<br>```New-AuthenticationPolicy -Name "Baseline - Block Basic Auth Protocols" -AllowBasicAuthAutodiscover:$False -AllowBasicAuthActiveSync:$False -AllowBasicAuthImap:$False -AllowBasicAuthMapi:$False -AllowBasicAuthOfflineAddressBook:$False -AllowBasicAuthOutlookService:$False -AllowBasicAuthPop:$False -AllowBasicAuthPowershell:$False -AllowBasicAuthReportingWebServices:$False -AllowBasicAuthRpc:$False -AllowBasicAuthSmtp:$False -AllowBasicAuthWebServices:$False```<br><br>```#Set default authentication policy for the organization```<br>```Set-OrganizationConfig -DefaultAuthenticationPolicy "Baseline - Block Basic Auth Protocols"```<br><br>```#Disable SMTP Basic auth at the transport layer```<br>```Set-TransportConfig -SmtpClientAuthenticationDisabled:$True```  | [Exchange Administrator](https://learn.microsoft.com/en-us/entra/identity/role-based-access-control/permissions-reference#exchange-administrator) Entra ID role| [MS Learn](https://learn.microsoft.com/en-us/exchange/clients-and-mobile-in-exchange-online/disable-basic-authentication-in-exchange-online#modify-authentication-policies) | 

By executing steps above, you're effectively adding another layer of defense - service-side legacy authentication blocking (1st layer) + conditional access policy blocking (2nd layer).

![342914334-3a6fe75e-2691-4d4a-a7fb-bad2cd75ebfd](https://github.com/lucas-ko/MicrosoftCloudNotes/assets/58331927/e4040744-ce6e-4f61-aa31-0301b1fadfdb)

When it comes to IT security controls, you must be prepared for control failures and remember: **_"Two is one, and one is none!"_** & **_"An ounce of prevention is worth a pound of cure."_** 

## Tip #3 - Policies do not apply to headless identities (service principals, managed identities) by default.

This is somehow a repeat from my previous [article](https://github.com/lucas-ko/MicrosoftCloudNotes/blob/main/EntraID-AppManagementPolicies.md#why-proper-governance-over-authentication-methods-for-applications-and-service-principals-is-important). But it's worth repeating :smile:

**Conditional Access policies, regardless of how well thought out they are, in a tenant with Entra ID P1/P2 licensing without any add-ons don't apply to service principals.**

To partially fix that:
- License your tenant with appropriate amount of Entra ID Workload Identities Premium licenses.
- Design, configure and apply policies (with limited configuration options) to service principals associated with applications you've created in your tenant.

As of now, you can't apply conditional access policies to govern third party application and service principal access - you must risk accept it. 

## Tip #4 - Conditional Access Evaluation - strict location enforcement - not enabled by default!

You might have implemented set of conditional access policies enforcing controls based on network location signals.<br>
Example scenarios include - requiring additional assurance (e.g. MFA) for authentication requests originating from outside of defined IP address ranges or simply blocking access outside of these ranges.

You might be convinced that by defining these network-based constraints, your externally accessible cloud resources are secured from unauthorized access outside defined network address ranges. 
However, it's not the case by default.

**If you haven't properly configured your network locations in Entra ID and didn't enable certain options in your conditional access policy configuration, all of that allows for the scenario where tokens sent to your cloud service (e.g. Exchange Online) can originate from outside of approved network location and still be accepted by that service ([source](https://learn.microsoft.com/en-us/entra/identity/conditional-access/concept-continuous-access-evaluation#exception-for-ip-address-variations-and-how-to-turn-off-the-exception)).**<br>

**This frequently happens in token theft scenarios, where attackers attempt to reply stolen access/ID tokens from unauthorized locations.<br>**

However, if you correctly define all IP address ranges of your clients[^2] and enable **'strictly enforce location policies'** session control in CA policies relying on network signals, you gain almost real-time revocation of these tokens. Strict location enforcement offers extra protection and considerable risk reduction against token replays outside of authorized network boundaries.
[^2]:As seen by Entra ID and resource providers - not a trivial task!

>[!NOTE]
>Only CAE capable clients and resources support this functionality. Compatibility list currently includes: Exchange Online, SharePoint Online, Microsoft Teams, Microsoft Graph and most of first party native and web clients.

1/ Define network locations (don't use countries - they don't work with CAE)<br>
![342677674-28753766-1cf0-42a0-8278-a5922b230707](https://github.com/lucas-ko/MicrosoftCloudNotes/assets/58331927/9c5d91ff-692c-425d-95e6-693c5c3e937a)

>[!CAUTION]
>Before you enable **'strictly enforce location policies'** in conditional access policy **you must ensure that all IP addresses from which your users can access Microsoft Entra ID and resource providers are included in the IP-based named locations policy**. Otherwise, you will block your users.

2/ Include locations in policy configuration<br>
![342712908-b5b24a66-8307-4234-a4f5-2cfde310b654](https://github.com/lucas-ko/MicrosoftCloudNotes/assets/58331927/751af88d-002f-41d5-a117-f616eac0a121)

3/ Enable strict location enforcement<br>
![342713118-3afe9d58-c0c3-4ad6-81cf-2eb94d4783a2](https://github.com/lucas-ko/MicrosoftCloudNotes/assets/58331927/12741a28-d28e-4e1a-bf23-53261d062ca0)

---
All work is licensed under a [Creative Commons Attribution 4.0 International License][cc-by].

[![CC BY 4.0][cc-by-image]][cc-by]

[cc-by]: http://creativecommons.org/licenses/by/4.0/
[cc-by-image]: https://i.creativecommons.org/l/by/4.0/88x31.png
[cc-by-shield]: https://img.shields.io/badge/License-CC%20BY%204.0-lightgrey.svg
