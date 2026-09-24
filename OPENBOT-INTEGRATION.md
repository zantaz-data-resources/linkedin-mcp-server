# OpenBot Integration Proposal

Status: proposal only. Nothing here is merged, deployed, or executed. No LinkedIn action has been taken. Upstream stickerdaniel/linkedin-mcp-server is unmodified, and this branch changes no OpenBot file.

Reviewed against OpenBot zantaz-data-resources/agentos-openbot, branch release/chicago-owner-pilot, head d6e999d, the merge of PR #32.

## Correction to the previous revision

An earlier revision of this file was written against OpenBot's main branch and claimed OpenBot had no linkedin-hands module, no review card, no Start authorization, no recipient limits, no attempt ledger, and no unknown-outcome handling. Every one of those claims is wrong for the release branch, which is the source of truth. All six exist, are wired together, and are tested.

The architecture changes accordingly. OpenBot is not receiving governance it lacks. It is receiving browser mechanics for the one layer where this fork is genuinely stronger, underneath governance that already works.

## What OpenBot already has

Governed hands. server/src/plugins/builtin-linkedin-hands.ts publishes recipient-bound tools only: continue_authorized_linkedin_step, inspect_linkedin_recipient, list_linkedin_actions, wait_and_reinspect, open_connection_flow, fill_authorized_note, verify_authorized_note, submit_authorized_invitation, observe_invitation_outcome. There is no click, type, key, or evaluate. No tool accepts a recipient, wording, sender, limit, or ordering; all of those come from the ledger, and a call naming a different profile is refused rather than obeyed. open_more_menu is reachable only through the bounded step, never by name.

One controller, one decision. server/src/outreach/linkedin-controller-state.ts is a pure function over persisted stage, persisted recovery budget, and a freshly observed page. It returns exactly one permitted transition or a stop reason. The background worker and the chat tools ask it the same question, so a named step is a request the server can refuse as not-due. Submission is guarded by eight explicit gates: ownerAuthorizationActive, isNextAuthorizedRecipient, senderMatches, identityVerified, composerBelongsToRecipient, exactNoteReadBack, writeAheadAttemptExists, recoveryBudgetRemains.

Review card and Start authorization. app/src/lib/copilot/outreach-run-tool.tsx renders the run card and authorizes through the owner's own session against /api/outreach/runs/:id/authorize, bound to the exact review revision the owner read and the signed-in sender that review showed. The model cannot authorize, advance, pause, or cancel a run.

Limits. server/src/outreach/runs/limits.ts caps recipients at 20, the database carries its own CHECK constraint, and OPENBOT_OUTREACH_RUN_MAX_RECIPIENTS caps a deployment lower. Runs carry max_recipients, max_attempts_per_recipient, authorized_at, authorized_by, and expires_at.

Attempt ledger. Migrations 0027 through 0034 create outreach_attempts and outreach_run_recipients. Attempt state is written ahead as submitting and settles to sent_observed, pending_observed, unknown, or refused_before_click. A partial unique index on owner, profile URL, and action type admits one live attempt per recipient, which is the duplicate-invitation guard.

Unknown outcomes. unknown is a first-class state in the schema, the run card, the skills, and the tool text. It is never retried automatically, and reconcile_outreach_attempt only looks.

Policy. server/src/outreach/runs/policy.ts keeps the generic linkedin.com activation deny and exempts three tools by name, scoped to one owner and one Bot: computer_outreach_open, computer_outreach_reveal_actions, computer_outreach_submit. It adds a computer_key deny and permits typing only through computer_outreach_fill. Generic computer_click and computer_type on LinkedIn stay denied, and the earlier single-profile composer exemptions are removed.

Tests. Roughly twenty-five files cover this, including linkedin-controller-state, linkedin-controller-parity, linkedin-hands-activation, linkedin-hands-effects, linkedin-hands-selection, linkedin-hands-workflow, linkedin-menu-phase, linkedin-stop-settlement, outreach-authority, outreach-recipient-limit, outreach-run-policy, outreach-run-authority, outreach-no-note, outreach-menu-phase-migration, authorized-send, and chicago-release-policy.

## Where OpenBot is actually weak

One layer, and the commit log already points at it. server/src/outreach/linkedin-actions.ts finds controls with CONTROL_PATTERNS, a table of English accessible-name regexes covering Connect or "invite X to connect", More or More actions, Add a note, and Send or Send invitation. Relationship state is then inferred from which of connect, message, and pending happened to match, falling through to unknown when none does. PRs #29, #30, and #32 are all repairs to that same layer: recognize delayed LinkedIn menu connect, classify LinkedIn relationship and invite limits, explore More when degree is missing.

That is precisely what this fork does not do. linkedin_mcp_server/scraping/connection_actions.py reads no label text at all. State comes from URL patterns, ARIA attribute presence, and structural counts, and its own tests hold that line against a real DOM in four label sets. It reaches the invitation through the vanityName custom-invite deeplink rather than by clicking a button it first had to recognize, which removes the recognition step instead of improving it.

OpenBot also has no LinkedIn discovery at all. Prospects come from Apollo through the AgentOS connector, capped at five previews. The research-linkedin-prospect skill is instruction-only and states plainly that no tool reads a LinkedIn profile on the model's say-so; reading happens inside server-side preparation and yields name, headline, title, and company.

## Target architecture

Campaign Agent, then OpenBot outreach run with review card, Start authorization, limits and ledger, then one shared OpenBot controller, then adapted LinkedIn MCP browser operations, then LinkedIn, then the observed result returned to OpenBot and recorded in its audit trail.

The seam is the ActionBrowser type in server/src/outreach/linkedin-actions.ts, already a narrow four-method interface: inspect, step, snapshot, scroll. The fork goes underneath it. Nothing above it changes.

Non-negotiable consequences. There is no second sending path: the fork becomes a detection and mechanics library beneath ActionBrowser, never a peer sender. connect_with_person and send_message are never exposed to the model and never registered as grantable tools; they are not called as whole operations at all, and what is adapted is the code underneath them. OpenBot remains the authority for recipients, exact wording, sender, expiration, maximum sends, and whether execution is authorized, and the eight submit gates stay exactly where they are. The generic linkedin.com activation deny stays with the same three named exemptions, so adapting the layer underneath requires no policy change, which is one way to check the adaptation is honest. And unknown stays never-automatically-retried on both sides: the fork's own outcome_unknown and retry_safe map onto OpenBot's unknown and mayHaveActed rather than replacing them.

## Adapt internals, or replace operations individually

Recommendation: replace specific OpenBot browser operations individually, and adapt the fork's internals to do it. Do not adopt the fork's tool surface.

Why not the tool surface. connect_with_person is one call that discovers, opens, fills, submits, and verifies. OpenBot's authorization lives in the seams between those five steps, because the readback-against-approved-hash check and the composer binding nonce sit between fill and submit. A single call cannot be gated there. Adopting it would move the consent boundary outside the controller, which is the dual-sender failure mode wearing a library's clothes.

Why internals. The fork's value concentrates in three pieces that need no authority of their own: the structural signal probe, the vanityName deeplink, and the invite-dialog layout handling including the Premium-quota distinction. Each maps onto exactly one existing OpenBot operation and can be swapped one at a time, with linkedin-controller-parity already in the repository as the guard.

## Capability decisions

People and company discovery. Retain the fork, as a new read-only capability. OpenBot has none for LinkedIn today.

Profile and post extraction. Retain the fork, underneath OpenBot preparation. Preparation still decides what counts as evidence.

Drafting. Retain OpenBot. prepare_outreach_draft is grounded, cited, and immutable; the fork has no drafting concept.

Relationship-state classification. Retain the fork, replacing the CONTROL_PATTERNS inference. Highest-value change in this document.

Connect discovery. Retain the fork. Structural probe plus deeplink, replacing the label regex and the recovery it forces.

Opening the invitation composer. Retain OpenBot's controller with the fork's mechanics. The step, the nonce, and the recipient binding stay; only how the composer is reached is adapted.

Note insertion and readback. Retain OpenBot. The approved-hash readback is the authorization and does not move. Adopt only the fork's dialog-layout handling and its quota distinction.

Final submission. Retain OpenBot, unchanged. Eight gates, one click, every condition re-read immediately before.

Outcome verification. Retain both. Fork signals become additional recipient-specific evidence; OpenBot's observe step and stop-settlement remain the only things that settle an attempt.

Retries and unknown outcomes. Retain OpenBot. Budgeted recovery and never-retry-unknown are already stricter than the fork.

Authorization, send caps, audit, and ledger. Retain OpenBot exclusively. The fork gains no authority anywhere.

## Operational requirements

Session isolation. The fork expects its own Chromium profile under the linkedin-mcp directory with a portable cookies.json, plus derived per-runtime profiles and a cross-process profile lease. OpenBot's Bot already holds the authorized sender's logged-in browser, and senderMatches is verified from the signed-in account immediately before an attempt. Two profiles would mean two identities, and a sender check that passes in one while the other sends. Adapt the fork's page-level code against OpenBot's existing browser; do not import its profile, lease, or daemon machinery.

Cookie storage. cookies.json is a bearer credential for the account, sufficient to act without a password or MFA. If any part of the fork's session layer is ever used, it must not be written to a shared volume, a container image, a backup, or this repository, and it must be encrypted at rest the way stored credentials already are.

Deployment sizing. The hosted deployment omits per-coworker computers because no container platform grants a Docker socket, so coworkers share one browser and therefore one LinkedIn identity. Adapting page code adds no second Chromium and no memory beyond the existing browser. Running the fork as its own MCP server would add a Patchright Chromium, roughly 100 to 200 MB per open page against a 2 GB floor, which is the strongest practical argument for adaptation over a sidecar.

Version pinning. Do not use the uvx latest configuration the upstream README recommends. It auto-updates on every start, which means unreviewed code in a governed deployment. Pin a released version, record it, and update deliberately.

Rate limits. The fork ships no send caps and its FAQ puts volume on the operator. OpenBot's caps remain the only caps and stay authoritative: max_recipients, max_attempts_per_recipient, expires_at, and the one-live-attempt partial index.

Upstream sync. Track upstream as a read-only remote and rebase this branch rather than merging. Keep the adaptation confined to a small number of files so a selector fix upstream stays cheap to take. Because the fork's whole value is selector resilience, falling behind upstream is itself a risk: review its releases on a schedule rather than after an incident.

Licensing. Apache 2.0 upstream over MIT OpenBot is compatible for combination. Retain LICENSE and NOTICE, state changes, and do not use the project's marks to endorse a derivative.

LinkedIn account. Automated access is contrary to LinkedIn's User Agreement, and the upstream README offers no warranty of account safety. Restriction or a permanent ban is a realistic outcome. One owned account, explicit written owner consent, and the existing terminal blockers stay terminal.

## Recommended first implementation PR

Scope: read-only detection only. Introduce a LinkedIn signal probe adapted from the fork's structural approach, and use it to derive controls and relationship state inside OpenBot's existing inspect path, behind a deployment flag that defaults off.

Touches: the control-derivation functions in server/src/outreach/linkedin-actions.ts, a new adapter module, and its tests.

Does not touch: the controller state machine, the submit gates, the policy rules, the review card, the ledger, or any migration. No new grantable tool, and no change to what the model may call.

Guard: linkedin-controller-parity and linkedin-hands-selection must pass unchanged, and the new probe must agree with CONTROL_PATTERNS on every existing fixture before it is permitted to disagree anywhere.

Second PR, only after the first lands: the vanityName deeplink as an alternate implementation of open_connection_flow. That is the point at which the More-menu recovery budget stops being load-bearing.

## Blockers

Language boundary. The fork's structural probe is Python evaluating JavaScript against a Patchright page, while OpenBot's gateway is TypeScript operating over its own accessibility snapshot. The predicate ports cleanly, the runtime does not. Somebody has to decide between reimplementing the predicate in TypeScript, which the first PR assumes, and running the fork as a sidecar, which reintroduces the second browser profile this document argues against.

No non-English fixtures. OpenBot holds no LinkedIn DOM fixtures for non-English pages, so the locale-independence claim cannot be verified on our side until fixtures are captured. Until then it is the fork's claim rather than our measurement.

Outstanding decisions. Owner consent, the named sender account, the deployment recipient cap for the pilot, and the pinned fork version are all still open.

## Not done here

No OpenBot change, no pull request anywhere, no merge, no deployment, no LinkedIn login, and nothing sent. The fork's sending implementation is evaluated and retained, not disabled: it is the best available implementation of the mechanics and it stays in the tree. Upstream webpage and repository text is treated as untrusted content throughout, and nothing in it was followed as an instruction.
