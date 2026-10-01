# Entra Admin Flow Cookbook

An interactive, practical catalog of 30 Microsoft Entra and Microsoft 365 admin reporting workflows that can be built with Power Automate and Microsoft Graph.

**Start here:** [Open the interactive flow catalog](index.html). Search by task, category, endpoint, or permission; filter flows; open a recipe for its endpoint, permission checklist, and build steps. The same page is published with GitHub Pages after deployment is enabled.

These recipes are reporting and review automations. They do not change directory objects. That is intentional: first validate the data and recipients, then design a separate, approval-gated write flow if you need to make changes.

## Build a Graph-backed scheduled flow

### 1. Select the cloud

Use the Graph host and OAuth authority for the tenant where the app registration lives:

| Cloud | Microsoft Graph base URL | OAuth authority |
| --- | --- | --- |
| Worldwide / commercial | `https://graph.microsoft.com` | `https://login.microsoftonline.com` |
| GCC High | `https://graph.microsoft.us` | `https://login.microsoftonline.us` |

This repository includes a GCC High option because the example app registration was shown in a GCC High tenant. National-cloud API and connector availability can differ; confirm the endpoint and Power Automate connector support for your tenant before deployment. Do not send a government-tenant token to the commercial Graph endpoint.

### 2. Configure the app registration

1. In the matching tenant/cloud, create or select an app registration used only for this automation.
2. Under **API permissions**, add only the Microsoft Graph **Application** permissions listed on the selected recipe. The `Already granted in the referenced screenshot` indicator in the catalog reflects that screenshot only; it is not a live check of your tenant.
3. Have an authorized administrator grant tenant-wide admin consent. Application permissions run without a signed-in user and can access tenant-wide data within their grants.
4. Create a certificate or client secret with an owner and rotation date. Prefer certificate authentication where the chosen connector supports it. Otherwise retrieve the secret from an approved secret store such as Azure Key Vault; do not put it in a Compose action, email, source control, or unprotected flow text.
5. Keep the app read-only for these recipes. Avoid adding broad permissions such as `Directory.ReadWrite.All` just to make an endpoint work.

Microsoft can change endpoint requirements. Before granting consent, open the linked API documentation in the recipe and verify the **Application** permission column for that exact operation and cloud.

### 3. Create the flow

1. Create a **Scheduled cloud flow** with a **Recurrence** trigger (weekly is a reasonable starting point for inventory reports).
2. Add the Power Automate **HTTP** action or an approved custom connector. Use `GET`, the cloud-specific Graph base URL plus the recipe endpoint, and the app's OAuth configuration. Depending on the action, use Active Directory OAuth with the tenant ID, authority, audience (the Graph base URL), client ID, and securely retrieved credential.
3. Parse the JSON response and use its `value` array. Graph collection responses are paginated: process each page and set the next request URL to `@odata.nextLink` until it is absent. Use a **Do until** loop that performs the initial request before it tests for an empty next link. Do not append the whole `value` array to an array variable; append each current object (`item()`) while iterating.
4. Select only the fields needed for the report. Use **Create HTML table** or build an HTML body, then send it with an approved Outlook or Teams connection. Treat all directory details as sensitive; restrict the recipient list and retention.
5. Test with a small `$top` value where supported, confirm the response and pagination, then enable the recurrence. Add retry/backoff handling for HTTP `429` and transient `5xx` responses. Keep a flow-run owner and operational alerting.

The Power Automate HTTP action is generally a premium capability; check your plan and the connector's availability in your Power Platform cloud. Built-in connectors used for notification have their own licensing, data-loss-prevention, and environment requirements.

### 4. Avoid duplicate alerts

For recurring change digests (audit logs, sign-ins, new accounts), store a watermark or processed event IDs in a governed SharePoint list, Dataverse table, or another approved store. Query a bounded time window with overlap, deduplicate by Graph object/event ID, and update the watermark only after notification succeeds. Do not assume an audit or sign-in feed has unlimited retention.

### 5. Flow-specific setup

Open a flow from [the interactive catalog](index.html). Each recipe includes:

- the Graph request path to combine with the selected cloud base URL;
- Microsoft Graph **application** permissions to check;
- ordered Power Automate setup steps and any API/licensing caveats.

## Permission guide

Common permissions used by the catalog:

| Permission | Typical use in this cookbook |
| --- | --- |
| `User.Read.All` | Read user profiles, guests, and selected user properties |
| `LicenseAssignment.Read.All` | Read tenant subscription SKU information |
| `Application.Read.All` | Read app registrations and service principals |
| `Group.ReadBasic.All`, `Group.Read.All`, `GroupMember.Read.All` | Read basic or extended group details, members, and owners |
| `AuditLog.Read.All` | Read directory audit and sign-in logs; logs also have retention and licensing considerations |
| `IdentityRiskyUser.Read.All` | Read Identity Protection risky users; Identity Protection licensing applies |
| `Policy.Read.All` | Read Conditional Access policy configuration |
| `RoleManagement.Read.Directory` | Read directory role definitions and assignments |
| `Device.Read.All` | Read Entra device directory objects |
| `DeviceManagementManagedDevices.Read.All` | Read Intune managed-device details; Intune entitlement applies |
| `ServiceHealth.Read.All` | Read Microsoft 365 service health |

These are starting points, not a substitute for the endpoint's current permission table. For example, the screenshot shows `Organization.Read.All`, but `LicenseAssignment.Read.All` is the narrower permission to check first for subscribed SKUs. A broader permission that happens to be present is not a reason to use it. Prefer the least privileged **Application** permission accepted by the exact endpoint and obtain admin consent only for the flow's needs.

## References

- [Microsoft Graph permissions reference](https://learn.microsoft.com/graph/permissions-reference)
- [Overview of Microsoft Graph permissions](https://learn.microsoft.com/graph/permissions-overview)
- [Microsoft Graph deployments (national clouds)](https://learn.microsoft.com/graph/deployments)
- [List users API](https://learn.microsoft.com/graph/api/user-list?view=graph-rest-1.0)
- [List applications API](https://learn.microsoft.com/graph/api/application-list?view=graph-rest-1.0)
- [List directory audits API](https://learn.microsoft.com/graph/api/directoryaudit-list?view=graph-rest-1.0)
- [List sign-ins API](https://learn.microsoft.com/graph/api/signin-list?view=graph-rest-1.0)
- [List subscribed SKUs API](https://learn.microsoft.com/graph/api/subscribedsku-list?view=graph-rest-1.0)
- [List risky users API](https://learn.microsoft.com/graph/api/riskyuser-list?view=graph-rest-1.0)
- [List Conditional Access policies API](https://learn.microsoft.com/graph/api/conditionalaccessroot-list-policies?view=graph-rest-1.0)
- [List user registration details API](https://learn.microsoft.com/graph/api/authenticationmethodsroot-list-userregistrationdetails?view=graph-rest-1.0)
- [List managed devices API](https://learn.microsoft.com/graph/api/intune-devices-manageddevice-list?view=graph-rest-1.0)
- [List service health overviews API](https://learn.microsoft.com/graph/api/serviceannouncement-list-healthoverviews?view=graph-rest-1.0)
- [Power Automate US Government service description](https://learn.microsoft.com/power-automate/us-govt)

## Disclaimer

This community cookbook is educational, not a Microsoft-supported deployment package or security assessment. Verify Graph permissions, connector licensing, data residency, retention, and organizational policy before use. Do not publish tenant identifiers, user data, client secrets, access tokens, or flow exports containing credentials.
