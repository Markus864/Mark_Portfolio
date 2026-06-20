# ADR-0004 — Service-account vs user-delegated authentication

- **Status:** Accepted
- **Decision drivers:** correct identity model, least privilege, auditability, user control

## Context

The assistant integrates with external services, and those integrations need
identity. There are two fundamentally different models, and choosing the wrong one
creates either a security problem or a functionality gap:

- **Service-account (machine) identity.** The integration authenticates *as
  itself* — a non-human principal with its own credentials and its own
  permissions. Right for machine-owned resources: a shared dataset, an
  organization's storage bucket, an internal API, a monitoring system.
- **User-delegated identity.** The integration acts *on behalf of a specific
  person*, against that person's own account, with that person's consent — the
  OAuth authorization-code model that yields a refresh token. Right for
  personal-scope resources: someone's own mailbox, calendar, documents, or media
  account.

Using a service account to reach a user's personal data is both wrong and often
impossible; using a user's delegated token for machine-to-machine automation
couples an automated system to one human's session and over-grants access.

## Decision

**Support both models behind the integration layer, choose per integration by who
owns the resource, and narrow privileges at runtime regardless of what was
granted.**

1. **Service accounts for machine-owned resources.** Where the resource belongs to
   the system or the organization, the integration uses a dedicated machine
   identity with its own least-privilege credentials, resolved from the secret
   store (see [ADR-0002](0002-tiered-secret-storage.md)).

2. **User-delegated OAuth for personal-scope resources.** Where the assistant must
   act on an individual's own account, it uses the authorization-code flow once,
   interactively, to obtain a long-lived refresh token, then mints short-lived
   access tokens as needed. The refresh token is stored in the secret store and is
   revocable by the user at any time from their account's permissions page.

3. **Runtime scope narrowing — the key discipline.** A broad grant at consent time
   is *not* treated as broad authority at call time. Even when a provider only
   offers a coarse scope, the runtime enforces a tighter boundary in code (for
   example, constraining file access to a single designated workspace folder even
   though the granted scope is account-wide). The grant is the ceiling; the
   runtime sets a lower floor.

4. **Consent and least privilege everywhere.** Request the *minimum* scopes that
   make the feature work; document each scope and why it is needed; and pair any
   destructive action with the framework's consent gate regardless of which auth
   model is in play.

## Consequences

**Positive**
- The identity model matches reality: machines act as machines, the assistant acts
  for a person only where that is the correct model and only with explicit
  consent.
- Runtime scope narrowing limits blast radius even when a provider's scopes are
  coarser than desired — a real, common situation — so an over-broad grant does
  not become over-broad behavior.
- Users retain control: a delegated grant is visible and revocable on their side,
  and revocation cleanly disables the affected capability.
- Credentials of both kinds live in the tiered store, never in source.

**Negative / trade-offs**
- Two auth paths are more to implement and document than one — accepted, because
  collapsing them would force an incorrect identity model on half the
  integrations.
- User-delegated flows require a one-time interactive consent step and ongoing
  token refresh handling; this is encapsulated in a provisioning step and the
  integration adapter so day-to-day operation is automatic.
- Runtime scope narrowing must be implemented and tested per integration — the
  cost of doing least privilege properly, and well worth it.

**Net:** picking the auth model by resource ownership, requesting minimal scopes,
and enforcing a tighter boundary in code than the grant allows gives a correct,
least-privilege, user-respecting identity story across every integration.
