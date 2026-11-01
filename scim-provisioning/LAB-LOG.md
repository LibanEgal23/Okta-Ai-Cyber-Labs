# SCIM Lab — Running Log (actual steps taken)

## Phase 1 — Catalog app attempt (SCIM 2.0 Test App (Header Auth))
1. Applications → Browse App Catalog → added "SCIM 2.0 Test App (Header Auth)"; Sign-On Options left at defaults → Done. Renamed label to "SCIM Okta LAB". (01-app-added.png)
2. Got SCIM target from https://scim.dev (SCIM Playground). Free plan = shared token `scim_free_access_token_playground`, base URL `https://api.scim.dev/scim/v2`, storage "Simulated — non-persistent, changes reset after every request". Signing up did not give a private key on the free plan.
3. Provisioning → Integration → Enable API integration → Base URL + API Token.
   - Plain token → 401 Unauthorized (error-02-unauthorized.png)
   - `Bearer scim_free_access_token_playground` → "verified successfully" (02-api-verified.png)
   - Lesson: scim.dev expects `Authorization: Bearer <token>`; the catalog app sends the token field as-is, so the prefix must be typed in.
4. Provisioning → To App → enabled Create Users, Update User Attributes, Deactivate Users → Save. (03-to-app-settings.png)
   - Mistake made & fixed: unchecked "Create Users" itself instead of only the sub-option "Set password when creating new users". The sub-option only appears when Create Users is checked.
5. Assignments → Assign to People → Shelly Cluff (test user) → profile mapping form auto-filled userName/givenName/familyName → Save and Go Back → Done. (04a-assign-form.png)
6. Result: 400 Bad Request — "Invalid data!" from remote server. (error-01-bad-request.png)
   - Disabling "Set password when creating new users" did NOT fix it.
   - Fixing token (Bearer) did NOT fix it → auth fine, payload rejected.
   - Conclusion: catalog app's user payload rejected by scim.dev. Decision: switch to custom SCIM app (Okta support article's documented method for scim.dev).
7. Cleanup: unassigned Shelly; app → Active ▼ → Deactivate → Delete (Okta requires deactivation before delete).

## Phase 2 — Custom SAML app with SCIM
1. Applications → Create App Integration (05a) → SAML 2.0 → App name (05b)
2. SAML settings: SSO URL and Audience URI both `https://scim.dev/`; other fields default (Name ID Unspecified, app username = Okta username). (06-custom-app-saml-settings.png)
3. Feedback: "This is an internal app that we have created" → Finish. (06b-internal-app-feedback.png)
4. General → App Settings → Edit → Provisioning: SCIM → Save. (07-scim-enabled.png)
5. Next: Provisioning → Integration → base URL, userName, all actions, HTTP Header auth, Bearer token → Test Connector Configuration.
6. Provisioning → Integration → Edit opens the "SCIM Connection" form (SCIM version 2.0 fixed). Default Authentication Mode is Basic Auth — must change to HTTP Header. (08-blank-scim-connection-form.png)

### Design decision: HTTP Header (Bearer) instead of OAuth 2.0
- scim.dev free plan provides a static bearer token; HTTP Header mode sends `Authorization: Bearer <token>` on every call.
- Okta's OAuth 2.0 mode needs authorize URL, token URL, client ID/secret and an auth flow — extra moving parts while debugging.
- Production note: OAuth 2.0 preferred (short-lived, auto-refreshed tokens; leaked token expires). Static tokens are common in SaaS SCIM APIs but must be rotated manually.
- Possible future lab: redo SCIM with OAuth 2.0.
7. Filled form: base URL, userName, all 5 actions, HTTP Header. (08a-integration-form.png)
   - Test Connector → 401 Unauthorized. (error-03-custom-app-unauthorized.png)
   - Cause: the custom app's Authorization field has a built-in, greyed-out "Bearer" prefix. Typing "Bearer <token>" produced "Bearer Bearer <token>".
   - Fix: enter only the token in this field. (Opposite of the catalog app, where "Bearer " had to be typed manually.)
   - Token-only value also returned 401 on Test Connector. Investigating (hidden whitespace / scim.dev shared token / save-then-test).
   - RESOLVED: retyped token by hand (only token, built-in Bearer prefix) → "Connector configured successfully". Likely cause of the repeat 401: hidden characters from copy/paste. (08b-connector-test.png)
   - Features detected: User Import, Import Profile Updates, Create Users, Update User Attributes, Push Groups, Import Groups. (Deactivate Users not listed in detection — enabled separately under To App.)
8. Saved integration → To App / To Okta appeared. To App → Edit → enabled Create Users, Update User Attributes, Deactivate Users; Sync User Password left off → Save. (10-to-app-enabled.png)
   - Note: custom app has no "Set password when creating new users" sub-option (catalog app did) — fewer fields sent.
9. Assigned Shelly to custom app → SAME 400 "Invalid data!" (error-04-custom-app-invalid-data.png)
   - Both catalog and custom apps fail identically with auth OK → issue is the user payload vs scim.dev free simulated plan, not app type.
   - Next diagnostics: (a) different test user, (b) scim.dev's own sample POST via Send, (c) Profile Editor mappings for extra/invalid attributes.
10. Unassigned Shelly; assigned test user Kobe Bryant (username = email, kobebryant24@yopmail.com) → NO error. (12-user-assigned-success.png)
   - Root cause (hypothesis, user's observation): Shelly's Okta username (shelly@efficientoffice.com) differs from her primary email (a gmail address). Mismatched username/email values produced a payload scim.dev rejected as "Invalid data". User whose username and email match provisions cleanly.
   - Lesson: before blaming the connector, test with a second user to separate profile-data problems from configuration problems.
   - Second clean user: Christina Agluinda (cagluinda@efficientoffice.com, username = email) assigned → no error. Confirms the fix pattern. (12b-second-user-assigned.png)
11. System Log (View Logs) review (13-system-log-provisioning.png):
   - Kobe Bryant: "Push new user to external application" SUCCESS → "Successfully pushed new user account to app" SUCCESS → "Sync user in external application" SUCCESS. Fully provisioned.
   - Christina Agluinda: "Add user to application membership" SUCCESS → one "Push new user" SUCCESS, then a second "Push new user" FAILURE (Bad Request / Invalid data!) and "Sync user" FAILURE. Assignments tab showed no red icon, but the log shows a partial failure.
   - Revised root cause: not simply username ≠ email (Christina's match). Common thread of failures = pre-built demo users (@efficientoffice.com) — likely an extra populated profile attribute in those profiles that scim.dev's free simulated plan rejects. Kobe (minimal profile) succeeds. To confirm later: Profile Editor → app mappings, compare Kobe vs Christina profiles.
   - Lesson: the Assignments tab can look clean while the System Log shows failures — always verify in the log.
12. Mover test: Kobe last name → "Bryant-Test". (14-system-log-update.png)
   - "Update user profile for Okta" SUCCESS; "Updated user application property" SUCCESS (Okta side updated).
   - "Push user's profile to external application" FAILURE: "No user returned for user <scim id>".
   - Cause: scim.dev free plan storage is "Simulated — non-persistent, changes reset after every request". The user Okta created no longer exists when Okta looks it up by SCIM id to push the update. Expected limitation of the test target, not an Okta misconfiguration. Okta did attempt the correct mover action.
   - This likely also explains Christina's second push/sync failure.
   - Production equivalent: a persistent SCIM server (real SaaS app or paid scim.dev) would return the user and accept the PATCH/PUT.
13. Leaver test: unassigned Kobe → "Remove user's application membership" SUCCESS. (15-system-log-deactivate.png, 15a-assignments-after-tests.png)
   - Before publishing, my public IP address was blacked out in the System Log screenshots.
