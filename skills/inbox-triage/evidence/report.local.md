# Inbox Triage Local Evidence Report

## Summary

`inbox-triage` reads a bounded inbox packet, classifies messages, drafts one
safe reply, and stops before sending by emitting a gated send proposal. The
package includes a deterministic `run.mjs` runner, so reviewers can execute the
implementation without relying on an agent-only path.

## Safety Boundary

- No private mailbox access is required.
- Fixtures are synthetic.
- The skill never sends, archives, deletes, unsubscribes, or mutates messages.
- Outbound email is represented only as `gated_send_proposal` with
  `requires_approval: true`.
- `operator_policy.auto_send: true` is refused before drafting.
- Drafting is bounded by supplied sender/operator topics, optional allowed
  intents, and allowed/forbidden commitments.

## Harness Coverage

The Docker/Linux `runx harness` verifies sealed execution and receipt generation
for these declared cases:

- `inbox-triage-happy-path-drafts-gated-reply`
- `inbox-triage-stops-on-missing-sender`
- `inbox-triage-stops-on-missing-body`
- `inbox-triage-refuses-auto-send`
- `inbox-triage-ready-on-digest-only`
- `inbox-triage-stops-when-commitments-not-allowed`

Semantic output checks are recorded separately in `evidence/local-smoke.json`.
Those checks assert the emitted schema, decisions, draft/no-draft behavior,
expected stop fields, happy-path classification labels, and gated send proposal
status.

## Local Verification

- `runx --version`: `runx-cli 0.6.6`
- Node.js in Docker harness: `v18.20.4`
- `runx harness /work/skills/inbox-triage --json --receipt-dir <temp_receipt_dir>`:
  passed
- Harness cases: `6`
- Harness assertion errors: `0`
- Semantic smoke cases: `6`
- Semantic smoke status: `passed`
- `runx doctor skills/inbox-triage --json`: success with zero diagnostics
- Receipt ids:
  `sha256:96936c9e66b2f9bbf72e1b7229fd53c6aa1fc2faa77051dd348339b57c1983ae`
  `sha256:360a053a5e0de01edffd3be6bccfc2202c052c5a456704b08ac755a6a389329a`
  `sha256:3b2c812578868b48fbc36dab87448eb5f6b4c0d4aa0bf49f6aec96247b1ba66f`
  `sha256:fe07522bdf96de3a5bb0e8634a4e1ed2213f53a793ecf8280bc76474b40f6086`
  `sha256:feb78b596c7681d0410ff549b5f71f03018d8a4efd46eff86f95db117db5fa63`
  `sha256:8d98771ef2fff8bc27bb6ea974cc80507c4bda35da1b36e318a816c45ea7efa2`

## Output Evidence

- Emitted packet schema: `runx.inbox.triage.v1`
- Classification labels:
  - `msg-001`: `needs_reply`, `time_sensitive`
  - `msg-002`: `digest`, `no_reply`
  - `msg-digest-001`: `digest`, `no_reply`
- Draft output: the reply acknowledges the sender, defers the final risk update
  until operator review, and avoids final approval.
- Stop conditions:
  - missing sender returns `needs_more_evidence`
  - missing message body returns `needs_more_evidence`
  - auto-send returns `refused`
  - digest-only returns `ready` with no draft
  - disallowed commitments return `needs_more_evidence`
- Gated send proposal: `requires_approval: true`,
  `approval_gate: send-as.explicit-operator-approval`, and
  `status: proposed_not_sent` on the happy path.

The Windows native harness currently fails before skill assertions because the
runx receipt store attempts to sync a directory handle on Windows. The same
failure reproduces on an existing runx skill, while `runx doctor
skills/inbox-triage --json` reports zero diagnostics. The Docker/Linux harness
therefore provides the local pre-submit harness evidence.

## Send-As Composition

The skill composes with send-as by producing a proposal containing recipient,
subject, body reference, and approval gate. A separate send-as runner or human
operator must verify authority and approve the proposal before any outbound
effect.

## Pending Public Submission Fields

The following fields are intentionally still pending because they require
public GitHub, runx registry, or Frantic account actions:

- `public_url`
- `source_url`
- `pr_url`
- raw `x_yaml`
- raw `skill_md`
- hosted `verification_json`
- public dogfood receipt and `runx verify` verdict
