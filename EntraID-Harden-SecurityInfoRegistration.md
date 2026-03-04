
## Introduction - why do we need to protect the authentication factor registration process?
I believe we can all agree that relying on the password alone isn't a viable authentication strategy anymore - if it ever truly was.<br>
As such, enforcing multi-factor authentication for user sign-ins shall be a critical priority for organizations of all sizes.<br>
Before rolling out strong authentication policies across the organization, it's essential to ensure all users within the tenant can securely register and manage their authentication methods. <br>These methods will play a key role during interactive sign-in scenarios that require strong authentication.<br>
This post explores how Microsoft Entra ID can help you manage the risks associated with the authentication method registration process, without compromising usability.

## Let's define the outcomes.
**To guide our design decisions, let's define the context of an imaginary company.**
- All corporate devices are Entra ID joined and managed by Intune.
- The company allows Microsoft Authenticator to be installed and used on unmanaged mobile devices.
- The strategic direction of the company is to adopt phishing-resistant authentication methods. While not enforced tenant-wide yet, the company initially aims to protect access to critical resources, including the security info registration portal.

**To complement already established context, let's consider a few simplified user stories. These will help us further with taking right design choices.**
- As an end user, I want a straightforward and seamless experience when registering strong authentication factors, either for the first time or when I need to make changes, so that I can easily comply with organizational policies during my sign-ins.
- As an IT security engineer responsible for the security of my tenant, I want to minimize the risks and configurations that do not generate enough friction for an attacker, so that I can have a high assurance only legitimate users are able to register or manage their authentication methods. 
- As an IT service desk personnel, I want a fast and simple troubleshooting process and recovery for end users who cannot access their authentication factors anymore, so that I can be effective at my job.
- As an IT engineer, I want the all security policies to be deployed without GUI, ideally using automation, so that I can be more efficient and have greater control over the configuration changes impacting that area.

## Policy creation.
Let's get to work.

One of the solutions supporting the above requirements is a single Entra ID conditional access policy, targeting security info registration user action, and requiring a specific authentication strength or specific device state, i.e. marked as compliant by Microsoft Intune.

As a result, we can arrive at the following policy definition:
 
- **Assignments**
  - **Include**: All users
  - **Exclude**: All B2B user types
- **Target Resources** -> User Actions -> Register security information
- **Conditions** -> None configured
- **Access Controls**
  - Grant
    - Require authentication strength (passkey or one-time temporary access pass)<br>
      **OR**
    - Require device to be marked as compliant
- **Session Controls**
  - Sign-in frequency -> 1h

**Benefits of such approach:**
- **User-friendly experience**
  - Users on compliant devices can register by authenticating with "traditional" MFA like push notifications or OATH TOTP (using implicit, built-in protection implemented by Microsoft for security information registration portal).
  - Unmanaged devices require stronger methods like phishing-resistant passkey.
  - Initial bootstrap requires admin-issued one-time TAP, balancing security with accessibility.
- **Alignment with Zero Trust design principles**
  -  Explicit verification of both user and device.
  -  No reliance on “trusted” network locations.
- **Phishing resistance**
  - Most registration flows* are protected against phishing.<br> _(*- initial bootstrap excluded - temporary access pass provides no security assurances to combat attacker in the middle (AiTM) scenarios)_
- **Minimizing session token replay window**
  - Sign-in frequency requires full user reauthentication every hour, reducing the validity of already issued session tokens. 
- **Operational simplicity**
  - Single policy simplifies troubleshooting and debugging.

## Deployment.

Let's deploy and materialize that concept!<br>
Following code uses Microsoft Graph Powershell to achieve this.

```
Import-Module Microsoft.Graph.Identity.SignIns

Connect-MGGraph -Scopes "Policy.ReadWrite.ConditionalAccess"

#Define the authentication strength object allowing passkeys or one-time TAP
$AuthStrengthParams = @{
	displayName = "Passkey_or_OneTimeTAP"
	#Marks the authentication event compliant with this authentication strength with MFA claim.
	requirementsSatisfied = "mfa"
	#Configure Passkeys or one time TAP as required methods to be used.
	allowedCombinations = @(
	"fido2",
	"temporaryAccessPassOneTime"
	)
}

#Create the new authentication strength object
$AuthStrength = New-MgPolicyAuthenticationStrengthPolicy -BodyParameter $AuthStrengthParams

$CAPolicyParams = @{
	#Set the name for the CA policy. Use your defined naming convention.
	displayName = "CA0XX-Global-SecurityInfoRegistration: Protect security information registration"
	state = "disabled"
conditions = @{
	applications = @{
		#Expliclitly target "register security information" user action.
		includeUserActions = @("urn:user:registersecurityinfo")
	}
	users = @{
		includeUsers = @("All")
		#Set the exclusions - consider excluding breakglass accounts for your environment.
		excludeUsers = @("GuestsOrExternalUsers")
	}
}
grantControls = @{
	operator = "OR"
	#Require a compliant device grant.
	builtInControls = @(
		"compliantDevice"
	)
	#Associate the authentication strength object with the CA policy.
	authenticationStrength = @{
		Id = $AuthStrength.Id
	}
}
sessionControls = @{
	signInFrequency = @{
		#Configure sign-in frequency - 1h.
		value = 1
type = "hours"
		isEnabled = $true
	}
}
}

#Create the new conditional access policy object.
New-MgIdentityConditionalAccessPolicy -BodyParameter $CAPolicyParams
```

## Things to watch out for.

- Certain edge cases could be confusing for an end user  - for example,  registering security information using TAP and requirement to change password at the same time is highly confusing and aggressive sign-in frequency (e.g., every time) time just adds to the confusion.<br>
Therefore the sign-in frequency suggested in the policy example has been adjusted to be less aggressive to reduce friction without overly compromising security assurances.
<br><br>
- Password change at [https://mysignins.microsoft.com](https://mysignins.microsoft.com) uses security info and is subject to CA policy targeting it.
<br><br>
- Passkey creation from an unmanaged device requires admin-issued TAP. Assess such scenario agains your organization risk tolerance.
As a best practice, registering passkeys on unmanaged devices should be minimized, as organizations can't really assure desired security properties of biometrics and a complexity of a device PIN which ultimately safeguards access to the passkey.
<br><br>
- For security info registration interrupt mode, the authentication strength is evaluated differently – authentication strengths that target the user action of **Registering security info** override authentication strength requirements from other conditional access policies targeting **All resources**.
All other grant controls (such as **Require device to be marked as compliant**) from other conditional access policies in scope for the sign-in will apply as usual.

## Useful resources:

- [Combined MFA/SSPR registration modes](https://learn.microsoft.com/en-us/entra/identity/authentication/concept-registration-mfa-sspr-combined)
- [Session controls for combined registration](https://learn.microsoft.com/en-us/entra/identity/authentication/concept-registration-mfa-sspr-combined#session-controls-for-combined-registration)
- [How Conditional Access authentication strengths work](https://learn.microsoft.com/en-us/entra/identity/authentication/concept-authentication-strength-how-it-works#how-multiple-conditional-access-authentication-strength-policies-are-evaluated-for-registering-security-info)

-------------------------------------------------------------------------------------------
All work is licensed under a [Creative Commons Attribution 4.0 International License][cc-by].

[![CC BY 4.0][cc-by-image]][cc-by]

[cc-by]: http://creativecommons.org/licenses/by/4.0/
[cc-by-image]: https://i.creativecommons.org/l/by/4.0/88x31.png
