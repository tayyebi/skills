---
name: testing-compliance
description: Use when testing privacy, regulatory compliance, and audit-trail posture from the outside — cookie consent, data-subject rights (access, erasure, portability, rectification), PII exposure in API responses and error bodies, audit-log completeness and immutability, data-retention correctness, payment-card scope, and data minimization. A regulatory axis distinct from attack-based testing — verifies what the system does with personal data and whether it honors the rights it claims to.
---

# Privacy & Compliance Testing

Black-box verification of privacy and regulatory posture: what data is collected, how it's exposed, whether deletion and export actually work, and whether the audit trail is accurate and complete. This is not vulnerability hunting — a system can be secure and still fail GDPR/CCPA/PCI/SOC 2 requirements, and this skill tests that separate axis.

## When to use

- Before launch in a jurisdiction with data-subject-rights law (GDPR, CCPA, etc.), or periodically as a compliance check on a live system.
- Any time a new field of personal data is added, a new export/deletion flow ships, or an admin-facing audit log is introduced.
- Identify which frameworks actually apply (data-subject-rights regimes for regions you serve, payment-card handling if you touch cards directly, an audit-trail standard if you offer one contractually) before testing — don't test against requirements that don't apply to the system.
- This is testing of your own system's posture; treat any real customer PII you encounter with the same care the policy claims to provide — capture only what's needed to prove a finding, mask it in reports, don't retain it beyond the test.

## Core pattern

**1. Cookie consent & tracking (first visit, before login).** In a clean/incognito session, record every cookie set and every third-party network call made *before* any consent action. A consent banner should appear; only essential/functional cookies should be set pre-consent; rejecting consent must not block access to the service (consent has to be freely given, not coerced); the preference should persist across visits.

**2. Privacy policy content.** Confirm it names the data controller and a contact for privacy requests, states retention periods per data category, lists third-party processors, and explains how to exercise data-subject rights. A dead or missing policy is itself a finding.

**3. Right to access.** The account's own "me"/profile endpoint and any dedicated export feature should surface (or let the user download) essentially everything held about them: account details, activity/order history, billing history, support history, an audit trail of their own actions, and any consent/marketing preference records. A partial export — profile fields only, no history — is a common gap.

**4. Right to erasure.** Walk the actual deletion flow end to end: create a throwaway account, generate some activity (a support request, a resource), request deletion, then confirm login now fails for that identity, the reset flow gives the same generic response as an unknown identity (no re-enumeration of the deleted account), and — critically — that what happens to related records is correct rather than merely "gone": billing/financial records typically must be *retained* for legal/tax reasons rather than deleted, while records not subject to a legal hold should be deleted or the personal identifiers within them anonymized. Check (as an admin, if reachable) whether anything still displays the deleted user's identifying details.

**5. Right to portability & rectification.** Confirm the export format is machine-readable (JSON/CSV, not PDF-only) and covers data the user themselves provided. Confirm profile edits (name, email, address) actually persist, and that the old identifier stops working for login once changed.

**6. PII exposure surface.** Audit ordinary API responses for anything beyond what the current user should see: does a list endpoint ever include another user's data; does the "me" object expose internal-only fields (password hashes, raw role/permission flags); does a 404 on someone else's resource leak their name/email in the body; does the password-reset endpoint return an identical response for a known vs. unknown identity; do 4xx/5xx error bodies ever echo back sensitive request fields.

**7. Audit-trail completeness.** Enumerate the material actions a system like this should record — logins and failed logins, credential changes, admin actions on a customer's account (impersonation, suspension, policy changes), account deletion, and any privileged data-access action. Perform each, then confirm it appears in the audit log with actor, action, target, timestamp, and IP — and that a null/missing actor never occurs. Confirm the log itself cannot be altered or deleted, even by the most privileged role — an editable audit trail is not an audit trail.

**8. Data retention.** Map each data category to its expected retention basis (financial records against tax law, audit logs against a stated security-retention window, session data with no long-term retention need, deleted-account data anonymized but billing records kept). Then probe for *premature* deletion — e.g., deleting a resource should not cascade-delete the financial record tied to it.

**9. Payment-card scope.** If a payment processor is used, confirm card data never transits your own servers (card entry should post directly to the processor's hosted field/JS, not to your API) and that stored payment-method records expose only last-4/brand/expiry, never a full card number.

**10. Data minimization.** Confirm signup/registration requires only what's functionally necessary, and that the user object doesn't carry fields the user never provided and the product doesn't need.

## Pitfalls

- Treating a policy document as sufficient evidence — the flows have to be exercised, not just read.
- Confusing "deleted" with "correctly retained-and-anonymized" — financial/legal records surviving is often the *correct* behavior, not a bug; the bug is when personal identifiers survive alongside them unnecessarily.
- Missing indirect PII leaks — a resource deleted from the primary table but still displaying the original owner's name in a related record (a ticket thread, an audit entry).
- Skipping the "identical response for known vs. unknown identity" check on both login and reset flows — enumeration is a privacy issue as much as a security one.
- Assuming an audit log is immutable without testing whether the most privileged role can edit or purge entries.

## Checklist

- [ ] Pre-consent cookie/tracking audit; consent banner blocks non-essential cookies; rejection doesn't block service
- [ ] Privacy policy names controller/contact, retention periods, processors, and rights process
- [ ] Access request surfaces full data (account, activity, billing, support, own audit trail)
- [ ] Erasure flow tested end to end: login fails after, no re-enumeration, legal records retained, personal identifiers anonymized elsewhere
- [ ] Export is machine-readable and covers user-provided data; rectification persists and old identifier stops working
- [ ] No cross-user PII in list/detail responses, "me" object, 404 bodies, or error bodies
- [ ] Known-vs-unknown identity responses identical on reset/login
- [ ] Every material admin/security action appears in the audit log with actor/action/target/timestamp; log is immutable even to top role
- [ ] Retention mapped per data category; no premature cascade-deletion of retained records
- [ ] Card data never transits your servers; stored payment methods show only last-4/brand/expiry
- [ ] Signup requires only necessary fields; no unexplained fields on the user object
