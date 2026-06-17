---
layout: post
title:  "Setting Up SAML 2.0 Federation Between Microsoft Entra ID and Amazon WorkSpaces Personal"
date:   2026-06-17 11:00:00 +0200
categories: aws entra saml workspaces
---
A practical, debugging-driven walkthrough of integrating Microsoft Entra ID (formerly Azure AD) as the SAML 2.0 identity provider for an Amazon WorkSpaces **Personal** environment backed by AWS Directory Service. This post documents the working end-state configuration and, just as importantly, the dead ends and error messages encountered along the way so you can skip them.

> **Note:** All account IDs, tenant IDs, provider identifiers, usernames, and email addresses in this post are placeholders. Replace them with your own values.

---

## Architecture Overview

The setup federates browser-based authentication for WorkSpaces through Entra ID:

- **Identity provider:** Microsoft Entra ID (Office 365)
- **Service:** Amazon WorkSpaces Personal
- **Directory:** AWS Directory Service (AWS Managed Microsoft AD) linked to the WorkSpaces directory
- **Federation mechanism:** SAML 2.0, IdP-initiated

A critical early realization: **WorkSpaces uses an IdP-initiated SAML flow.** It does *not* generate a `SAMLRequest`. Entra issues the assertion on its own, and the WorkSpaces client registration is what kicks off the flow. This single fact explains a whole class of confusing errors.

Another key constraint: SAML 2.0 for WorkSpaces Personal is **only** available when the directory is managed through AWS Directory Service (Simple AD, AD Connector, or AWS Managed Microsoft AD). Directories managed directly by Amazon WorkSpaces use IAM Identity Center instead.

---

## Prerequisites

- An Entra ID tenant with an enterprise application you can configure
- An AWS account with permissions for IAM and WorkSpaces
- A WorkSpaces Personal directory backed by AWS Directory Service
- A provisioned WorkSpace assigned to a directory user
- The directory user's `sAMAccountName` (this matters enormously — see below)

---

## Step 1: Create the SAML Identity Provider in AWS IAM

In **IAM → Identity providers → Add provider**:

1. Choose **SAML**.
2. Give it a name (e.g. `Entra`). **Remember this name** — it is case-sensitive and referenced everywhere downstream.
3. Upload the **Federation Metadata XML** downloaded from your Entra enterprise application's SAML Certificates section.

> **Gotcha:** If you ever recreate this provider, AWS assigns it a **new internal identifier**, which changes the ACS URL (see Step 4). Keep the metadata current — if Entra's signing certificate rotates, re-upload the metadata or signature validation fails silently.

---

## Step 2: Create the SAML Federation IAM Role

Create an IAM role for SAML 2.0 federation. The role needs a **trust policy** (who can assume it) and a **permissions policy** (what they can do).

### Trust policy (final, hardened version)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::ACCOUNT_ID:saml-provider/Entra"
      },
      "Action": [
        "sts:AssumeRoleWithSAML",
        "sts:TagSession"
      ],
      "Condition": {
        "StringLike": {
          "SAML:aud": "https://signin.aws.amazon.com/saml*"
        }
      }
    }
  ]
}
```

Two things worth calling out:

- **`sts:TagSession` is required** alongside `sts:AssumeRoleWithSAML`. WorkSpaces SAML flows can send session tags, and omitting this action causes `Not authorized to perform sts:AssumeRoleWithSAML`.
- **Use `StringLike` with a trailing wildcard on `SAML:aud`.** A plain `StringEquals` on the exact audience string can deny the assume-role even when the assertion's audience visually matches — likely due to a trailing-slash or string-normalization difference. The wildcard absorbs that while still pinning the audience to the AWS sign-in endpoint.

> **Avoid:** A `"SAML:sub_type": "persistent"` condition unless you are certain your assertion's subject type is genuinely presented as persistent. Adding it prematurely produces `AccessDenied` on an otherwise-valid assertion.

### Permissions policy

The trust policy only governs *assuming* the role. The role also needs a permissions policy granting WorkSpaces streaming. A scoped example:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "workspaces:Stream",
      "Resource": "arn:aws:workspaces:REGION:ACCOUNT_ID:directory/DIRECTORY_ID",
      "Condition": {
        "StringEquals": {
          "workspaces:userId": "${saml:sub}"
        }
      }
    }
  ]
}
```

Make sure the **region** and **directory ID** match your actual directory. `${saml:sub}` resolves to the NameID value, so it must align with the WorkSpace's `userId`.

---

## Step 3: Configure the Entra Enterprise Application

In **Entra admin center → Enterprise applications → [your app] → Single sign-on (SAML)**.

### Basic SAML Configuration

- **Identifier (Entity ID):** `https://signin.aws.amazon.com/saml`
- **Reply URL (ACS):** `https://signin.aws.amazon.com/saml/acs/PROVIDER_ID`

> **Critical:** The Reply URL is **provider-specific**. AWS appends a unique identifier to the ACS URL (e.g. `.../acs/SAMLSPXXXXXXXX`). You can find this value as the **Sign-in URL** on your IAM identity provider's details page. Using the bare `https://signin.aws.amazon.com/saml` causes the Entra error `AADSTS50011: reply URL does not match`. If you recreate the IAM provider, this ID changes and must be updated here.

### Attributes & Claims

This is where most of the debugging happened. The final working claim set:

| Claim | Source |
|---|---|
| **NameID** (Unique User Identifier) | `ExtractMailPrefix(user.userprincipalname)` — format set to persistent |
| `https://aws.amazon.com/SAML/Attributes/Role` | Constant: `arn:aws:iam::ACCOUNT_ID:saml-provider/Entra,arn:aws:iam::ACCOUNT_ID:role/YourRoleName` |
| `https://aws.amazon.com/SAML/Attributes/RoleSessionName` | `ExtractMailPrefix(user.userprincipalname)` |

#### The single most important detail: NameID must match the directory `sAMAccountName`

For WorkSpaces Personal, AWS matches the assertion to the Active Directory user using the **NameID value**, which must equal the WorkSpaces username (the `sAMAccountName`).

If the directory user is `jane.doe` but Entra sends the UPN `jane.doe@example.com`, the match fails. The fix is the built-in **`ExtractMailPrefix()`** transformation, which strips the domain and emits just `jane.doe`.

> **Two separate editors:** The **NameID** and **RoleSessionName** claims are configured in different places. Fixing one does not fix the other. The match that matters for the directory lookup is the **NameID**.

#### Entering the Role claim as a constant

Entra builds claim values from attributes, but the `Role` claim needs a static two-ARN string. To enter a constant: set **Source** to **Attribute**, then type the literal ARN-pair string directly into the **Source attribute** field (no quotes). The format is `provider-ARN,role-ARN` — AWS accepts either order. The attribute name must be exactly `https://aws.amazon.com/SAML/Attributes/Role` (case-sensitive).

---

## Step 4: Configure the WorkSpaces Directory

In the **WorkSpaces console → Directories → [your directory] → SAML 2.0 settings**:

- **User Access URL:** the Entra IdP-initiated URL — the **User access URL** from your enterprise application's **Properties** page (the `myapps`/launcher URL), **not** the `/saml2` endpoint.
- **IdP deep link parameter name:** `RelayState` (the default; Entra uses this exact name).
- **Enable SAML 2.0 authentication:** checked.

> **Gotcha:** The `/saml2` Login URL shown in section 4 of the Entra SSO page ("Set up [app]") is the SP-binding endpoint. Pointing the User Access URL there triggers `AADSTS750054: SAMLRequest or SAMLResponse must be present`, because that endpoint expects an SP-initiated request that WorkSpaces never sends. Use the IdP-initiated launcher URL instead.

---

## The Debugging Journey: Errors and What They Meant

A chronological map of the errors hit, in case you hit the same ones.

### `AADSTS750054: SAMLRequest or SAMLResponse must be present`
The User Access URL pointed at the `/saml2` SP-binding endpoint. WorkSpaces is IdP-initiated and sends no `SAMLRequest`. **Fix:** use the Entra IdP-initiated launcher URL.

### `Your request included an invalid SAML response`
The assertion reached AWS but was rejected — typically a missing/malformed `Role` claim or a missing `NameID`. **Fix:** ensure the `Role` attribute is present and correctly named, and that a `NameID` exists.

### `RoleSessionName is required in AuthnResponse`
The `RoleSessionName` claim was missing. **Fix:** add `https://aws.amazon.com/SAML/Attributes/RoleSessionName`.

### `Not authorized to perform sts:AssumeRoleWithSAML`
The assertion was valid but the **trust policy** refused it. Causes encountered, in order:
- Missing `sts:TagSession` action.
- A `SAML:sub_type` / audience condition that evaluated to false.
- Ultimately resolved by using `StringLike` on `SAML:aud` and dropping the `sub_type` condition.

### `AADSTS50011: reply URL does not match`
The Entra Reply URL didn't match the provider-specific ACS URL. This surfaced after recreating the IAM provider (which changed the ACS ID). **Fix:** set the Reply URL to the current `.../acs/PROVIDER_ID`.

### `An error occurred while launching your WorkSpace` (client-side)
This appears **after** authentication fully succeeds — it is a WorkSpace runtime/connectivity issue, not a SAML problem. Confirmed by widening the IAM permissions to `workspaces:*` on `*` with no change. Things to check: reboot the WorkSpace, pull client diagnostic logs, verify streaming ports (4172 / 4195) are open, and confirm the client/protocol (PCoIP vs. WSP/DCV) match.

---

## Debugging Tips That Paid Off

**Decode the SAML assertion.** The fastest way to see what Entra is actually sending. Capture the `SAMLResponse` form field from the POST to `signin.aws.amazon.com/saml` (browser dev tools → Network, or the SAML-tracer extension) and base64-decode it. The `SAMLResponse` is base64 but not deflate-compressed, so a plain decode yields readable XML. Inspect `<NameID>`, `<Audience>`, and the `Role`/`RoleSessionName` attributes.

**Check CloudTrail in us-east-1.** `AssumeRoleWithSAML` is a global STS event and lands in **us-east-1**, regardless of your WorkSpaces region. If you can't find it in your regional console, look there. The event confirms whether STS resolved the user, role, and provider — and whether it returned `AccessDenied`.

**Isolate variables with a permissive policy.** When stuck on authorization, temporarily widening the trust or permissions policy proves whether the problem is policy-side or elsewhere. (Tighten it back afterward — see the hardened trust policy above.)

**Test from the client, not the browser.** For WorkSpaces Personal, the browser IdP-initiated flow only **registers** the client. Actual desktop sign-in must originate from the WorkSpaces client application.

---

## Final Working Configuration Summary

| Component | Value |
|---|---|
| Entra Entity ID | `https://signin.aws.amazon.com/saml` |
| Entra Reply URL | `https://signin.aws.amazon.com/saml/acs/PROVIDER_ID` |
| NameID source | `ExtractMailPrefix(user.userprincipalname)` (persistent) |
| Role claim | `arn:aws:iam::ACCOUNT_ID:saml-provider/Entra,arn:aws:iam::ACCOUNT_ID:role/YourRoleName` |
| RoleSessionName source | `ExtractMailPrefix(user.userprincipalname)` |
| WorkSpaces User Access URL | Entra IdP-initiated launcher URL |
| IdP deep link parameter | `RelayState` |
| Trust policy actions | `sts:AssumeRoleWithSAML`, `sts:TagSession` |
| Trust policy condition | `StringLike` on `SAML:aud` = `https://signin.aws.amazon.com/saml*` |

---

## Key Takeaways

1. **WorkSpaces SAML is IdP-initiated** — use the Entra launcher URL, not the `/saml2` endpoint.
2. **NameID must equal the directory `sAMAccountName`** — use `ExtractMailPrefix()` to strip the UPN domain.
3. **Include `sts:TagSession`** in the trust policy.
4. **Use `StringLike` for `SAML:aud`** to avoid brittle exact-match denials.
5. **The provider-specific ACS URL changes** if you recreate the IAM provider — keep Entra's Reply URL in sync.
6. **Decode the assertion and check us-east-1 CloudTrail** — they remove the guesswork.
7. **Authentication success ≠ launch success** — a post-auth launch error is a WorkSpace connectivity issue, separate from federation.
