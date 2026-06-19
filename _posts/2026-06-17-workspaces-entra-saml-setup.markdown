---
layout: post
title:  "Setting Up Amazon WorkSpaces Personal with SAML 2.0 Authentication via Microsoft Entra ID"
date:   2026-06-17 11:00:00 +0200
categories: aws entra saml workspaces
---
A step-by-step guide to federating Amazon WorkSpaces Personal sign-in through Microsoft Entra ID (formerly Azure AD), for a directory backed by AWS Directory Service (AWS Managed Microsoft AD). This walks through the full configuration and flags the caveats that are easy to miss.

> All account IDs, tenant IDs, provider identifiers, domains, usernames, and emails below are placeholders. Replace them with your own values.

---

## How the flow works (read this first)

Understanding the model saves a lot of confusion later. Authentication happens in **two distinct stages**:

1. **SAML federation (`SAML_IAM`)** — Entra authenticates the user's identity to AWS, an IAM role is assumed, and the WorkSpaces client registers and reaches the directory.
2. **Directory sign-in (`WARP_DRIVE` / Unified Sign-In)** — the user authenticates to the Active Directory itself (with their AD password) to unlock the actual desktop session.

So **SAML gets you *to* your WorkSpace; the AD password logs you *into* it.** Expecting a single password-free SSO will surprise you — the second prompt is by design. (If you want to eliminate it, see Certificate-Based Authentication at the end.)

Two hard prerequisites for this to be possible at all:

- WorkSpaces **Personal** SAML is only supported when the directory is managed through **AWS Directory Service** (Simple AD, AD Connector, or AWS Managed Microsoft AD). Directories managed directly by Amazon WorkSpaces use IAM Identity Center instead.
- The WorkSpace must already exist, be `AVAILABLE`, and be assigned to a directory user.

---

## Step 1: Create the SAML identity provider in AWS IAM

1. IAM → **Identity providers** → **Add provider**.
2. Provider type: **SAML**.
3. Name it (e.g. `Entra`). This name is **case-sensitive** and referenced in the role and claims — keep it consistent.
4. Upload the **Federation Metadata XML** from your Entra enterprise application (you'll create that in Step 3; you can come back and upload it, or do Step 3 first).

After creation, open the provider and note its **assigned ACS URL** — it looks like:

```
https://signin.aws.amazon.com/saml/acs/SAMLSPXXXXXXXXXXXX
```

You'll need this exact value for the Entra Reply URL.

> **Caveat:** If you ever delete and recreate this provider, AWS assigns a **new** ACS identifier. Every place that references the old one (notably the Entra Reply URL) must be updated, or sign-in breaks with a reply-URL mismatch.

---

## Step 2: Create the SAML federation IAM role

Create a role for SAML 2.0 federation with a trust policy and a permissions policy.

### Trust policy

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

Key points:

- **`sts:TagSession` is required** in addition to `sts:AssumeRoleWithSAML`, because the assertion carries a `PrincipalTag` (the email). Omitting it causes `Not authorized to perform sts:AssumeRoleWithSAML`.
- Use **`StringLike`** with a trailing `*` on `SAML:aud`. A strict `StringEquals` on the exact audience can deny assume-role even when the audience visually matches; the wildcard avoids that while still pinning the audience to the AWS sign-in endpoint.
- Only add a `SAML:sub_type` = `persistent` condition if you are certain the NameID format is genuinely persistent. Adding it otherwise produces `AccessDenied`.

### Permissions policy

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

> **Caveat (this one is a classic time-sink):** the `DIRECTORY_ID` in the resource ARN must be your **real** WorkSpaces directory ID (e.g. `d-XXXXXXXXXX`), and `REGION` must match where your directory lives. A placeholder or wrong directory ID lets authentication succeed but fails the launch later. Get the directory ID from the WorkSpaces console.

---

## Step 3: Configure the Entra enterprise application

In **Entra admin center → Enterprise applications → New application → Create your own application** (non-gallery), then open **Single sign-on → SAML**.

### Basic SAML Configuration

- **Identifier (Entity ID):** `https://signin.aws.amazon.com/saml`
- **Reply URL (ACS):** the **provider-specific** ACS from Step 1:
  ```
  https://signin.aws.amazon.com/saml/acs/SAMLSPXXXXXXXXXXXX
  ```

> **Caveat:** Use the provider-specific `.../acs/SAMLSP...` URL, **not** the bare `https://signin.aws.amazon.com/saml`. The bare form triggers `AADSTS50011: reply URL does not match`.

### Attributes & Claims

Configure exactly these. The two that matter most for the directory match are the **NameID** and **PrincipalTag:Email**.

| Claim | Source |
|---|---|
| **Unique User Identifier (NameID)** | Transformation: `ExtractMailPrefix(user.userprincipalname)` |
| `https://aws.amazon.com/SAML/Attributes/Role` | Constant: `arn:aws:iam::ACCOUNT_ID:saml-provider/Entra,arn:aws:iam::ACCOUNT_ID:role/YourRoleName` |
| `https://aws.amazon.com/SAML/Attributes/RoleSessionName` | `user.userprincipalname` (or `ExtractMailPrefix(...)` to match the username) |
| `https://aws.amazon.com/SAML/Attributes/PrincipalTag:Email` | `user.mail` |

Notes on each:

- **NameID** must equal the user's `sAMAccountName` (the WorkSpaces username). If your Entra UPN is `user@domain.com` but the AD logon name is just `user`, use `ExtractMailPrefix(user.userprincipalname)` to strip the domain so the NameID becomes the bare username.
- For the **Role** claim, set Source to **Attribute** and type the literal ARN-pair string directly into the Source attribute field (no quotes). The name must be exactly `https://aws.amazon.com/SAML/Attributes/Role` (case-sensitive). AWS accepts either ARN order.
- **PrincipalTag:Email** must carry the user's email and `user.mail` must be populated in Entra. If the Entra account's Email field is blank, this claim is sent empty and the directory match fails.

> **The single most important matching rule:** WorkSpaces matches the assertion to the AD user on **two** fields — the **NameID** must equal the AD `sAMAccountName`, and the **PrincipalTag:Email** must equal the AD user's `mail`/`EmailAddress`. Both must match **exactly** (same case, same domain). A mismatch on either produces `USER_AUTH_FAILURE` at the broker.

Finally, **assign the user/group** to the enterprise application, and download the **Federation Metadata XML** to upload into the IAM provider (Step 1).

---

## Step 4: Configure SAML on the WorkSpaces directory

WorkSpaces console → **Directories** → select your directory → **Authentication / Edit** → enable **SAML 2.0**.

- **User Access URL:** the Entra **IdP-initiated** URL — the *User access URL* from the enterprise application's **Properties** page (the `myapps`/launcher URL). **Not** the `/saml2` endpoint.
- **IdP deep link parameter name:** `RelayState` (Entra's default; leaving it blank also works).

> **Caveat:** The `/saml2` Login URL shown in section 4 of the Entra SSO page is the SP-binding endpoint and will fail with `AADSTS750054`. Use the IdP-initiated launcher URL from the app's Properties page.

---

## Step 5: Verify the Active Directory user

Before testing, confirm the directory user is consistent and ready. On a domain-joined management instance:

```powershell
Get-ADUser -Identity "username" -Properties sAMAccountName,userPrincipalName,mail,EmailAddress |
  Select-Object sAMAccountName,userPrincipalName,mail,EmailAddress
```

Check that:

- `sAMAccountName` equals what your **NameID** sends.
- `mail` (and `EmailAddress`) equal what your **PrincipalTag:Email** sends — **exactly**, including domain and case. (A leftover/incorrect email here from earlier testing is a very common cause of `USER_AUTH_FAILURE`.)
- The password is set and valid, and **"User must change password at next logon" is unchecked** — otherwise the directory sign-in (Step 6's password prompt) fails.

---

## Step 6: Test the end-to-end sign-in

Test from the **WorkSpaces client application**, not the browser — for Personal, the browser IdP-initiated flow only registers the client; the actual desktop sign-in happens in the client.

1. Open the WorkSpaces client, enter your registration code.
2. The client redirects to Entra; sign in.
3. SAML federation completes and the client registers.
4. At the **Unified Sign-In** prompt, the username is pre-filled (from your NameID). Enter the **AD password**.
5. The desktop session launches.

---

## Caveats and things to check

A consolidated checklist of the non-obvious things:

- **Two-stage auth is normal.** SAML federates access; the AD password logs into the desktop. Expect both.
- **Match on both NameID and email, exactly.** `sAMAccountName` ↔ NameID and `mail` ↔ PrincipalTag:Email. Case and domain must match. This is the most common failure point.
- **`user.mail` must be populated in Entra**, or the email claim is empty.
- **Directory ID and region** in the role's permissions policy must be real and correct.
- **`sts:TagSession`** must be in the trust policy.
- **Provider-specific ACS URL** in Entra's Reply URL; it changes if you recreate the IAM provider.
- **Region support:** confirm your Region supports WorkSpaces Personal SAML before assuming it will work.
- **Transient network errors** (`The network connection was lost`, `NSURLErrorDomain Code=-1005`) at the `WARP_DRIVE` step are usually just connectivity blips to `ws-broker-service.<region>.amazonaws.com` — retry, and if persistent, test on a different network and confirm the WorkSpaces streaming ports/endpoints are reachable.

---

## Debugging tips

**Decode the SAML assertion.** The fastest way to see what Entra actually sends. Use the **SAML-tracer** browser extension and run the sign-in; it shows the decoded assertion XML. Check `<NameID>`, the `PrincipalTag:Email` attribute, `Role`, `RoleSessionName`, and `<Audience>`.

**CloudTrail for AssumeRoleWithSAML.** This is a **global** STS event — look in **us-east-1** regardless of your WorkSpaces region. The event shows whether STS resolved the identity/role/provider and the exact error if assume-role fails.

**WorkSpaces client logs.** These contain the specific error code at each stage (e.g. `USER_AUTH_FAILURE`, the `SAML_IAM` vs `WARP_DRIVE` step, network errors). They're located at:

- **Windows:** `%LOCALAPPDATA%\Amazon Web Services\Amazon WorkSpaces\logs`
- **macOS:** `~/Library/Application Support/Amazon Web Services/Amazon WorkSpaces/logs`

In the logs, the line `AuthProvider: SAML_IAM` is the federation step and `AuthProvider: WARP_DRIVE` is the directory password step — knowing which one fails tells you whether to look at SAML/IAM config or at the AD user/password.

---

## Final working configuration summary

| Component | Value |
|---|---|
| Entra Entity ID | `https://signin.aws.amazon.com/saml` |
| Entra Reply URL | `https://signin.aws.amazon.com/saml/acs/SAMLSPXXXXXXXXXXXX` |
| NameID source | `ExtractMailPrefix(user.userprincipalname)` |
| Role claim | `arn:aws:iam::ACCOUNT_ID:saml-provider/Entra,arn:aws:iam::ACCOUNT_ID:role/YourRoleName` |
| RoleSessionName source | `user.userprincipalname` |
| PrincipalTag:Email source | `user.mail` |
| WorkSpaces User Access URL | Entra IdP-initiated launcher URL |
| IdP deep link parameter | `RelayState` |
| Trust policy actions | `sts:AssumeRoleWithSAML`, `sts:TagSession` |
| Trust policy condition | `StringLike` on `SAML:aud` = `https://signin.aws.amazon.com/saml*` |
| Permissions policy | `workspaces:Stream` on the directory ARN (correct region + directory ID) |

---

## Optional: eliminate the second password prompt (Certificate-Based Authentication)

If the AD password prompt after SAML is undesirable, **Certificate-Based Authentication (CBA)** makes the desktop login passwordless: the directory trusts a certificate derived from the SAML identity instead of prompting for the AD password. CBA requires AWS Private CA and additional claims (such as `PrincipalTag:ObjectSid` and `PrincipalTag:UserPrincipalName`). It's a separate project, but it's the documented path to true single sign-on for WorkSpaces.
