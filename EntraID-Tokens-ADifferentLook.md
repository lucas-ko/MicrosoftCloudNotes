## Entra ID tokens and cookies - a different perspective.

No, I am not going to bore you to death with another article on how refresh tokens, access tokens and session cookies work and how their implementation in Microsoft Entra ID looks like.<br>
In this article, I would rather like to consider specific sub-types of refresh tokens and session cookies and under which circumstances can they be revoked (or not!).

### Tokens vs cookies - how do they differ?
Refresh tokens and session cookies are different types of long-lived tokens.<br> 
They are issued from a single root authentication event. Usually, it's an interactive authentication in the case of human operated identities.<br>
Key difference between them?<br>
- Refresh tokens are issued when interacting with applications using OAuth2 framework (i.e. mobile, desktop apps and web applications calling secured APIs in the background).<br>
- Session cookies are issued when interacting with, web (HTTP-based) applications. In this article, I am only considering session cookies issued by Entra ID to Microsoft first party web apps.<br>

### Differences within the family
Now, lets dive directly into their sub-types.<br>
We can distinguish password-based and non-password-based tokens/cookies. 

Password-based refresh tokens and session cookies are issued and tied to an interactive authentication that used password as part of the authentication flow.<br>
As you might infer, non-password-based tokens/cookies are issued and tied to an interactive authentication that didn't use password as part of the authentication flow.<br>
Examples include FIDO2 security key or passkey-based authentication, using Microsoft Authenticator passwordless phone-based sign-in, Temporary Access Pass, certificate based authentication, as well as SMS-based or recently added QR code sign-in.<br>

### Impact on incident response
All cool, but let's ask the important question - what happens if a Security Operations team, performing a response to a user identity compromise, decide to just reset the user's password. Does this action remediate the risk well enough?<br>
Unfortunately not - all non-password-based tokens and session cookies tied to the compromised user identity will stay alive and active. Only password-based post-authentication artifacts are revoked!<br>
However, if Security Operations team perform an admin revocation of user tokens - all kinds of tokens and cookies, regardless of their sub-type, are revoked.<br>
It is therefore imperative that SOC teams incorporate not only user password reset, but also admin revocation of refresh tokens, as part of their identity compromise response playbooks.<br>

Below table summarizes what happens to tokens/cookies after administrative actions:
| Action | Password-based token | Password-based session cookie | Non-password-based token | Non-password-based session cookie |
|---|---|---|---|---|
| Admin forces password reset | Revoked  | Revoked  | Valid | Valid |
| Admin revokes all refresh tokens | Revoked  | Revoked  | Revoked  | Revoked  |

### How to perform admin revocation of all tokens and session cookies?

PowerShell Graph SDK way (required directory roles needed neatly documented [here](https://www.azadvertizer.net/azentraidroleactions/microsoft.directory_users_invalidateallrefreshtokens.html), Graph API permissions [here](https://learn.microsoft.com/en-us/graph/api/user-invalidateallrefreshtokens?view=graph-rest-beta&tabs=http#permissions]):

```
Import-Module Microsoft.Graph.Beta.Users.Actions
Connect-MgGraph
Invoke-MgBetaInvalidateAllUserRefreshToken -UserId <UserPrincipalName or object ID>
```

ClickOps (GUI) way:<br>

![RevokeSessions](https://github.com/user-attachments/assets/22f26806-ea13-4d1f-a734-003830bfc64e)


### What about external (B2B) users? 
As an admin of a resource tenant, you can't revoke the tokens and session cookies issued to external users domiciled in another Entra ID tenant or externally federated organization. Such users must have their tokens revoked in their parent organization.

## Useful resources:

- [Refresh tokens in the Microsoft identity platform](https://learn.microsoft.com/en-us/entra/identity-platform/refresh-tokens)
- [Authentication methods reference claim in access tokens](https://learn.microsoft.com/en-us/entra/identity-platform/access-token-claims-reference#amr-claim)
- [Entra ID directory roles with invalidate all refresh tokens permission](https://www.azadvertizer.net/azentraidroleactions/microsoft.directory_users_invalidateallrefreshtokens.html)
- [MS Graph permissions - user: invalidateAllRefreshTokens](https://learn.microsoft.com/en-us/graph/api/user-invalidateallrefreshtokens?view=graph-rest-beta&tabs=http#permissions)

-------------------------------------------------------------------------------------------
All work is licensed under a [Creative Commons Attribution 4.0 International License][cc-by].

[![CC BY 4.0][cc-by-image]][cc-by]

[cc-by]: http://creativecommons.org/licenses/by/4.0/
[cc-by-image]: https://i.creativecommons.org/l/by/4.0/88x31.png
[cc-by-shield]: https://img.shields.io/badge/License-CC%20BY%204.0-lightgrey.svg

