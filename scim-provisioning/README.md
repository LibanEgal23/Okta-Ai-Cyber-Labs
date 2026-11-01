# Lab: Automated User Provisioning with SCIM 2.0 in Okta

> **Status:** 🚧 In progress
> **Platform:** Okta Workforce Identity Cloud (Admin Console)
> **Skills:** SCIM 2.0, lifecycle management (joiner / mover / leaver), app provisioning, group push, System Log troubleshooting

---

## 📌 Objective

Configure Okta to automatically **create, update, and deactivate** user accounts in a downstream application using **SCIM 2.0**. The goal is to remove manual account management and make Okta the source of truth for the user lifecycle.

## 🧠 Why This Matters

Without provisioning, IT creates accounts by hand in every app, and accounts often stay active after someone leaves. That's a security and audit risk. SCIM is the open standard (RFC 7643 / 7644) that lets an identity provider manage accounts in apps over a REST API. With SCIM:

| Lifecycle event | What Okta does over SCIM |
|---|---|
| **Joiner:** user assigned to the app | `POST /Users` creates the account |
| **Mover:** profile changes in Okta | `PATCH/PUT /Users/{id}` updates attributes |
| **Leaver:** user unassigned or deactivated | `PATCH /Users/{id}` sets `active: false` |
| **Group push** | `POST/PATCH /Groups` syncs group membership |

## 🏗️ Architecture

```
┌──────────────┐    SCIM 2.0 over HTTPS (REST + JSON)    ┌──────────────────────┐
│     Okta     │ ──────────────────────────────────────► │  SCIM app / server   │
│  (IdP, source│   Authorization: Bearer <API token>     │  /scim/v2/Users      │
│   of truth)  │ ◄────────────────────────────────────── │  /scim/v2/Groups     │
└──────────────┘          201 Created / 200 OK           └──────────────────────┘
```

## 🧰 Environment

| Item | Value |
|---|---|
| Okta org | _e.g. Okta Integrator Free Plan / trial org_ |
| SCIM app | _e.g. SCIM 2.0 Test App (Header Auth)_ |
| SCIM server / target | _fill in_ |
| Auth method | _e.g. HTTP Header (Bearer token)_ |
| Test user(s) | _e.g. scim.test1@yourdomain.com_ |
| Test group | _e.g. SCIM-Lab-Users_ |

---

## 🛠️ Steps

### 1. Add a SCIM-enabled app
- **Applications → Applications → Browse App Catalog**
- Search **SCIM 2.0 Test App (Header Auth)** → **Add Integration** → name it → **Next → Done**
- _Alternative (custom app):_ **Create App Integration → SAML 2.0**, then **General → App Settings → Edit → Provisioning: SCIM → Save**

📸 `screenshots/01-app-added.png`

### 2. Configure the API integration
- **Provisioning → Integration → Configure API Integration**
- Check **Enable API integration**
- Enter the **Base URL** (ends in `/scim/v2`) and the **API token**
- Custom app only: set the unique identifier to `userName`, check **Push New Users / Push Profile Updates / Push Groups**, and set the auth mode to **HTTP Header**
- **Test API Credentials → Save**

📸 `screenshots/02-api-verified.png`

### 3. Enable provisioning actions
- **Provisioning → To App → Edit**
- Enable **Create Users**, **Update User Attributes**, and **Deactivate Users** → **Save**

📸 `screenshots/03-to-app-settings.png`

### 4. Assign a test user (Joiner)
- **Assignments → Assign → Assign to People** → select the test user → **Assign → Save and Go Back → Done**
- Confirm the user appears in the SCIM app/server

📸 `screenshots/04-user-created.png`

### 5. Push a group
- **Push Groups → Push Groups → Find groups by name** → select the group → **Save**
- Confirm the push status shows **Active** and the group exists in the target

📸 `screenshots/05-group-pushed.png`

### 6. Test profile update (Mover)
- **Directory → People** → edit the test user's first or last name → **Save**
- Confirm the change shows up in the target app

📸 `screenshots/06-user-updated.png`

### 7. Test deprovisioning (Leaver)
- Unassign the user from the app (or deactivate the user in Okta)
- Confirm the user is set to `active: false` in the target

📸 `screenshots/07-user-deactivated.png`

### 8. Verify in the System Log
- **Reports → System Log** → search by the app name
- Review events such as `application.provision.user.push`, `application.provision.user.deactivate`, and `application.provision.group_push`

📸 `screenshots/08-system-log.png`

---

## ✅ Results

| Test | Expected | Result |
|---|---|---|
| API credentials verified | Green success banner | ⬜ |
| Assign user → account created | User exists in target | ⬜ |
| Push group → group created | Group exists, status Active | ⬜ |
| Update profile → attribute synced | Change reflected in target | ⬜ |
| Unassign user → account deactivated | `active: false` in target | ⬜ |
| Events visible in System Log | Success events logged | ⬜ |

## 🧯 Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| "Test API Credentials" fails | Wrong Base URL or token | Check the URL ends in `/scim/v2` with no trailing slash; regenerate and re-paste the token |
| 401 Unauthorized | Wrong auth mode or header format | Make sure the auth mode matches what the server expects (Bearer vs Basic) |
| User assigned but not created | **Create Users** not enabled under To App | Enable it, then use **Assignments → Retry/Provision** |
| Error about a duplicate user (409) | User already exists in the target | Delete the user in the target or match on `userName` |
| Group push stuck or errored | **Push Groups** not enabled in the integration | Enable it under the Integration settings and re-push |
| Can't find the cause | — | Check **Reports → System Log** and **Dashboard → Tasks** for the exact error |

## 💡 Key Takeaways

- _What I learned about SCIM and the provisioning lifecycle_
- _What broke and how I fixed it_
- _How this would scale in a real enterprise (e.g. HR-driven provisioning → Okta → SCIM apps)_

## 📚 References

- [Okta: Build a SCIM provisioning integration](https://developer.okta.com/docs/guides/scim-provisioning-integration-overview/main/)
- [RFC 7643: SCIM Core Schema](https://datatracker.ietf.org/doc/html/rfc7643)
- [RFC 7644: SCIM Protocol](https://datatracker.ietf.org/doc/html/rfc7644)
