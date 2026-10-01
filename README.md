# Entra Admin Flow Cookbook

An interactive catalog of 60 Microsoft Entra and Microsoft 365 admin reporting workflows for Power Automate and Microsoft Graph.

**Beginner?** Start with the [interactive catalog](index.html) and its **Beginner setup: app registration and HTTP** section at the top. Search or filter for a flow, open its details, copy its complete HTTP URI, then follow the same HTTP field instructions plus that recipe's own steps. The public site is [mspfe2019.github.io/entra-admin-flow-cookbook](https://mspfe2019.github.io/entra-admin-flow-cookbook/).

These cookbook recipes are for reporting and review. They do not make changes to users, groups, apps, devices, or policies.

## Beginner guide: build the HTTP request

The HTTP setup is the same for all 60 recipes. Only the URI and required application permission change from recipe to recipe.

### Before you start

Have the following values from an app registration in the same tenant and cloud as the data:

- **Directory (tenant) ID** — identifies your Entra tenant.
- **Application (client) ID** — identifies the app registration.
- **Graph Application permission** — open the recipe and add only the permission required by its endpoint under **API permissions → Add a permission → Microsoft Graph → Application permissions**. An administrator must grant admin consent.
- **Credential** — use an approved secure connection/certificate where supported. If using a client secret, use its **Value**, not its ID, and store/retrieve it from an approved vault. Do not paste secrets into a URL, email, Compose output, screenshot, or source code.

The permission chips in the catalog are guidance, not a live check. Always open that recipe's linked Microsoft API documentation and verify its **Application permissions** table before requesting consent.

### Create a scheduled flow

1. In Power Automate, choose **Create → Scheduled cloud flow**.
2. Name it (for example, `Guest inventory report`) and set a weekly schedule while testing.
3. Choose **New step**, search for **HTTP**, and select the HTTP action approved in your environment. It is often a Premium action; check your license and connector availability.
4. In the cookbook, open a recipe, select the correct cloud, and choose **Copy full endpoint**. Use that URL in the URI field below.

### Fill in the HTTP action

| Field | What to enter |
| --- | --- |
| **Method** | `GET` — each cookbook example reads data. |
| **URI** | Paste the complete URI copied from the recipe. It includes the correct Graph cloud hostname, `/v1.0/`, and the recipe's endpoint path and query. |
| **Headers** | If available, add `Accept` with value `application/json`. |
| **Queries** | Leave blank when the copied URI already contains query options after `?`. Do not add them a second time. |
| **Body** | Leave empty for a GET request. |
| **Authentication** | Choose **Active Directory OAuth** when using the standard HTTP action. Some approved connector versions may show **OAuth** or use a separate authenticated HTTP connector. |
| **Authority** | The sign-in hostname for the same cloud as your tenant (see below). |
| **Tenant** | Your Directory (tenant) ID. |
| **Audience** | The Graph base URL only, such as `https://graph.microsoft.com`. Do **not** include `/v1.0` or an endpoint path. |
| **Client ID** | Your Application (client) ID. |
| **Credential type / Secret** | Select **Secret** and supply the securely stored secret **Value** only if the connector requires it. Some connectors support a certificate or managed connection instead. |

If Authority or other fields aren't visible, look under **Show advanced options**. Do not add a manual bearer-token header when OAuth is configured to acquire the token.

| Tenant cloud | Full URI starts with | Authority | Audience |
| --- | --- | --- | --- |
| Worldwide / commercial | `https://graph.microsoft.com/v1.0/` | `https://login.microsoftonline.com` | `https://graph.microsoft.com` |
| GCC High | `https://graph.microsoft.us/v1.0/` | `https://login.microsoftonline.us` | `https://graph.microsoft.us` |

For example, the `/users?$select=id,displayName` recipe path becomes:

```text
https://graph.microsoft.com/v1.0/users?$select=id,displayName
```

For GCC High, use `https://graph.microsoft.us/v1.0/users?$select=id,displayName`. Do not mix a government tenant with the commercial Graph or authority host. Other national clouds can have different hosts. Check [Microsoft Graph deployments](https://learn.microsoft.com/graph/deployments).

> **Connector screens differ.** This guide describes the standard HTTP action using Active Directory OAuth. If your action does not show these fields, pause and check that exact connector's Microsoft documentation or ask your Power Platform administrator. Do not guess where to put an app secret.

### Test and troubleshoot

1. Select **Save → Test** and run the flow manually if available.
2. Open **Run history**, select the run, and expand the HTTP action. Status `200` means Graph accepted the request.
3. **401 Unauthorized:** check the tenant ID, Authority, Audience, Client ID, credential, and cloud; all must match the app registration.
4. **403 Forbidden:** check that the exact Graph Application permission is added and tenant admin consent was granted.
5. **404 Not Found:** check the cloud host, `/v1.0` path, endpoint spelling, and any `{placeholder}` values in the recipe.
6. **429 Too Many Requests:** Graph is throttling. Wait the `Retry-After` period and retry; do not rapidly loop the request.
7. Treat run-history output as tenant data. Never paste real user, tenant, credential, or audit information into public tickets or repositories.

### Read the response and create a report

Most list endpoints return a JSON object with a `value` array. The HTTP **Body** is the enclosing object, not the array itself.

1. Add **Parse JSON** after HTTP. For **Content**, choose the HTTP action's **Body**. Use **Generate from sample** with a small, sanitized Graph response to produce its schema.
2. Add **Apply to each**. For its input, choose the `value` array from Parse JSON. If the dynamic-content picker only offers Body, use **Expression** and enter `body('HTTP')?['value']` (replace `HTTP` with your action's actual name).
3. Inside the loop, `item()` means the current record. Append `item()` or its selected fields when building an array; do not append the whole `value` array inside the loop.
4. Use **Select** to choose only the output columns, then **Create HTML table** or an approved Outlook/Teams action. Restrict recipients and report retention.
5. A small number of requests return one object, not a `value` array (for example, the organization profile). In those cases, use the fields from Body directly and do not add an Apply to each.

### Recipe-specific URLs and extra requests

- Some recipe URIs contain tokens in braces, such as `{id}`, `{domainId}`, or `{UTC_START}`. These are instructions, not literal IDs: replace each with a real ID or a Power Automate expression before running.
- For `{UTC_START}`, use a date/time in UTC and URL-encode it. Example Power Automate expression for the prior 24 hours: `encodeUriComponent(formatDateTime(addHours(utcNow(),-24),'yyyy-MM-ddTHH:mm:ssZ'))`.
- When a recipe says to query something "for each" returned user, group, app, or service principal, add a **second HTTP action inside that record's Apply to each**. Insert the current record's ID in the second URI. Use the same Method and cloud-matched authentication values.
- Follow each recipe's **Power Automate build steps**. They describe what to filter, summarize, join, or send after the HTTP request.

### Pagination for larger results

Graph may split a long list into pages. If the response contains `@odata.nextLink`, there are more results.

1. Initialize a string variable `nextLink` to the first full URI copied from the recipe.
2. Add a **Do until** loop that stops when `equals(variables('nextLink'), '')`.
3. Inside the loop, run HTTP with `variables('nextLink')` as its URI. Process the current page's `body('HTTP')?['value']` array.
4. After processing the page, set `nextLink` to:

```text
coalesce(body('HTTP')?['@odata.nextLink'], '')
```

The first HTTP request must run before the loop tests for an empty next link. Let Graph provide the next-page URL; don't construct or edit that URL yourself. Set a suitable Do until iteration limit and honor throttling.

### Before enabling the schedule

Test a report with a restricted recipient, confirm all pages and expected columns, and store a watermark/processed event IDs for recurring change or sign-in digests to prevent duplicate notifications. Minimize personal data, restrict flow co-owners and run-history access, and define an owner for flow failures and credential rotation. Verify endpoint permissions, national-cloud support, retention, data-loss-prevention requirements, and Premium licensing for HTTP.

## Accessibility

The interactive catalog includes adjustable text size, high contrast, keyboard navigation, visible focus indicators, skip-to-main-content, screen-reader announcements, focus-managed dialogs, and reduced-motion support. These features are not a formal accessibility certification.

## Permission guide

Permissions below are examples and should be checked against each endpoint's current documentation. Prefer the least privileged Application permission that endpoint accepts; admin consent is generally required.

| Permission | Typical use |
| --- | --- |
| `User.Read.All` | Read user profiles, guests, and selected user properties |
| `LicenseAssignment.Read.All` | Read tenant subscription SKU information |
| `Application.Read.All` | Read app registrations and service principals |
| `Group.ReadBasic.All`, `Group.Read.All`, `GroupMember.Read.All` | Read groups, members, and owners |
| `AuditLog.Read.All` | Read directory audit and sign-in logs |
| `IdentityRiskyUser.Read.All`, `IdentityRiskEvent.Read.All` | Read Identity Protection risky users/detections |
| `Policy.Read.All` | Read Conditional Access and selected policy configuration |
| `RoleManagement.Read.Directory`, `RoleEligibilitySchedule.Read.Directory` | Read directory role definitions, assignments, and PIM eligibility |
| `Device.Read.All` | Read Entra device directory objects |
| `DeviceManagementManagedDevices.Read.All` | Read Intune managed-device details; Intune license applies |
| `ServiceHealth.Read.All` | Read Microsoft 365 service health |
| `Domain.Read.All` | Read verified domains and domain configuration |
| `AdministrativeUnit.Read.All` | Read administrative units |
| `AccessReview.Read.All` | Read access review configuration |
| `DelegatedPermissionGrant.Read.All` | Read delegated OAuth permission grants |

The screenshot referenced in the catalog showed `User.Read.All`, `Application.Read.All`, `Directory.Read.All`, and `Organization.Read.All` as granted Application permissions. That is not a live status check. For example, `LicenseAssignment.Read.All` is the narrower permission to check for the subscribed SKU endpoint instead of adding broader access.

## References

- [Microsoft Graph permissions reference](https://learn.microsoft.com/graph/permissions-reference)
- [Overview of Microsoft Graph permissions](https://learn.microsoft.com/graph/permissions-overview)
- [Microsoft Graph national-cloud deployments](https://learn.microsoft.com/graph/deployments)
- [Power Automate US Government](https://learn.microsoft.com/power-automate/us-govt)
- [List users API](https://learn.microsoft.com/graph/api/user-list?view=graph-rest-1.0)
- [List applications API](https://learn.microsoft.com/graph/api/application-list?view=graph-rest-1.0)
- [List groups API](https://learn.microsoft.com/graph/api/group-list?view=graph-rest-1.0)
- [List directory audits API](https://learn.microsoft.com/graph/api/directoryaudit-list?view=graph-rest-1.0)
- [List sign-ins API](https://learn.microsoft.com/graph/api/signin-list?view=graph-rest-1.0)
- [List subscribed SKUs API](https://learn.microsoft.com/graph/api/subscribedsku-list?view=graph-rest-1.0)
- [List domains API](https://learn.microsoft.com/graph/api/domain-list?view=graph-rest-1.0)
- [List service health overviews API](https://learn.microsoft.com/graph/api/serviceannouncement-list-healthoverviews?view=graph-rest-1.0)

## Disclaimer

This community cookbook is educational, not a Microsoft-supported deployment package or security assessment. Verify permissions, connector licensing, data residency, retention, and organizational policy before use. Never publish tenant identifiers, user data, client secrets, access tokens, or flow exports containing credentials.
