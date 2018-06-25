
Information stored in Active Directory defines "you" with the exception of built-in security groups.  This would be fine, although the number of security groups tends to get out of control, except the OpenId Connect standard only account for an access token providing "application level" security and an identity token with "slowly changing" personal inforamtion. 

The following is an unimplemented design that would provide system permissions to authenticated requestors 
and remove the responsibility for maintaining permissions from the authentication system as well as deal with "quickly changing" permission attributes. 

[iamge here]

## Whats the difference between Identity and a Permission?
Identity is immutable for the lifetime of a session.  An identity is made up of a series of claims that are individual pieces of identity information such as name, email, job title, hair color, or locale.  Personally I think of this type of information as "slowly changing",  you may get promoted or change your name but it doesn't happen all the time.  There are certain instances where this information can be very valuable to an application and permissions can be inferred, like your job title being a "Doctor vs a Nurse" but that concept usually doesn't get us all the way to where we want to go.  An example of a permission, sometimes called an entitlement, could be would be "Jeremy can edit the calendar in the CRM application".

## What problem are we trying to solve?
 * Permissions should live within a development lifecycle, just because I can decrypt credit cards in dev doesn't mean I should do it in production. Many organizations have a single Active Directory instance across all development environments or lifecycles which leads to an excess of security groups in AD (ProdCalendar, QACalendar, etc).  
 * It's hard to audit who had a permission on a certain date.
 * There are lots of permissions, but only a few are relevant for auditors so we usually provide very broad permissions report for audit which runs the risk of brining un-necessary items into audit scope. 
 * Operations teams, who usually grant permissions, have no concept of permission severity or intent.  A user creates a ticket that says "I need access to X", which needs to get translated into a specific permission name.  In my experience, this usually happens by "copying" the permissions off another users (usually manager) persmissions which can result in elevated or un-necessary permissions.  
 * We do our best to remove permissions, but most of the time we get them for at least 6 months (or life depending on internal audits).
 * Applications don't treat permissions as an attribute that can change and often store them with a long timeout or require a logout/login to refresh. 
 * Users can have access to many applications with many permissions.  I've seen single users with 25,000 individual permissions. 

We've been creating our own authorization stores attached to applications for years, which is great for application teams because it gives them control but problematic for security / compliance and other governance based functions.

## What about the information in an Identity/Access Token or the response from the UserInfo Endpoint?
These mechanisms, outlined in the [OpenID Connect Core 1.0][OpenID Connect Core 1.0] specification are designed to transmit claims or an extended set of claims, not permissions.  In fact the transportation of information, especially in http headers, from those sources can be problematic for at least the following reasons.

 * The "Single Responsibility Principal" tells us that seperating slowly changing and quickly changing attributes is a good thing. 
 * Permissions are almost always scoped to an application and should be accessed by it's specific namespace to cut down on collisions and the potential to inadvertently hijack another applications claim.
 * Permissions and business logic often overlap, a centralized permission storage mechanism would make it impossible for applications to force them to overlap. 
 * Tokens should stay small, browser length restrictions and bandwitdh are often limiting factors. Especially in high, throughput applciations.  
 * It's easy to add a claim to a token and unimaginably difficult to remove the same claim, especially if an application starts to crash after its removal.  Compatibility becomes a real issue. 

## A Sample Payload
This payload represents a specific user and their current permissions, I may add an extended properties field here to hold additional information.
~~~~~~
{
	"sub": "domain\\user"
	"PermissionScopes": [{
			"scope": "App1",
			"permissions": [{
					"permission": "App1CalendarRead",
					"expiration": "2018-06-25T14:35:45-04:00",
					"approvedBy": "domain\\supervisor"
				}, {
					"permission": " App1CalendarWrite",
					"expiration": "2018-06-25T14:35:45-04:00",
					"approvedBy": "domain\\supervisor"
				}
			]
		}, {
			"scope": "App2",
			"permissions": [{
					"permission": "App2Permission",
					"expiration": "2018-06-25T14:35:45-04:00",
					"approvedBy": "domain\\evp"
				}, {
					"permission": "App2Permission2",
					"expiration": "2018-06-25T14:35:45-04:00",
					"approvedBy": "domain\\evp"
				}
			]
		}, {
			"scope": "GlobalNamespace",
			"permissions": [{
					"permission": "CommonOrGlobalNamespacePermission",
					"expiration": "2018-06-25T14:35:45-04:00",
					"approvedBy": "domain\\supervisor"
				}, {
					"permission": "CommonOrGlobalNamespacePermission2",
					"expiration": "2018-06-25T14:35:45-04:00",
					"approvedBy": "domain\\ceo"
				}
			]
		}
	]
}
~~~~~~

A payload for a root permission should hold the name, description, an expriation timespan, and an approver.  

## Real-world hurdles
* Integration is an issue, many applications may have trouble finding resources to switch their authentication systsms so a synchronization should be utilized for compatibility. 
* Applications should request permissions on behalf of the logged in user, this will cut down the operational burden.  
* Signatures are sometimes required for permissions, that should happen in the operations website. 
* A mobile application would be an awesome way to get a push notification that you need to approve a permission. 
* Reporting by application, permission, or user for a period of time should show all changes to permissions.
* It's usually harder than it should be to get an organizational hierarchy, figuring out exactly who should approve permission requests is non-trivial.  
* You can't put EventGrid in a vNet, meaning these requests would travel outside a company network, this may be a non-starter for some companies. 
* Authobot is a bad name for this, bots are a real thing and this isn't a bot.  Naming is hard. 

## Questions to answer
* Are permissions public?  Do I care if people see the areas in which I have access?
* What about user impersonation?  Is this a specific permission created by and managed by speficic applications?
* Do all permissions need to be time-boxed?  Can something exist forever?
* Permissions are to be namespaced, should there be a global namespace for 2 applications that share some of the same permissions?
* Does this interact with Priveliged Identity Managmenet? Maybe a PIM Permission request initiates here?  I could go either way on this, the consumer of this product is an application not an administrator doing maintenance. 

[OpenID Connect Core 1.0]: http://openid.net/specs/openid-connect-core-1_0.html