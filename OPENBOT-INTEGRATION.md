# OpenBot Integration Proposal

Status: proposal only. Nothing here is merged, deployed, or executed. No LinkedIn action has been taken. Upstream stickerdaniel/linkedin-mcp-server is unmodified.

## Why this document exists

zantaz-data-resources/agentos-openbot has a Campaign Agent that is expected to research LinkedIn prospects and prepare outreach. Today it drives LinkedIn through the generic governed computer: computer_navigate, computer_click, computer_type, evaluated by the CEL action policy in tools/deploy/openbot.env.template and audited by the server gateway.

Two facts decide the whole design.

First, OpenBot has no LinkedIn sending implementation. There is no linkedin-hands module and no outreach controller in the repository. What exists is a generic browser gateway plus a hard deny. The hosted policy ships mode enforce with allow true and two deny rules, and server/tests/agentos-linkedin-deployment-policy.test.ts asserts that every activation on any linkedin.com host is refused regardless of coworker, button label, or Enter and Space keypresses. The Campaign Agent prompt in examples/fintech/agents.yaml instructs the model to stop before the final Connect or Send and hand control to a human, and tools/deploy/README.md states plainly that the pilot has no machine-enforced approval token for a final send.

Second, the MCP server does have a complete, tested sending implementation. connect_with_person and send_message carry real Connect discovery, note insertion, submit, and post-write verification, plus outcome_unknown and retry_safe semantics that OpenBot does not currently express anywhere.

So integration is not a merge of two senders. It is giving OpenBot's governance a capable pair of hands, and building the controls the brief assumes already exist.

## The unified path

Campaign Agent -> OpenBot orchestration, authorization, and audit -> LinkedIn MCP browser tools -> LinkedIn -> observed result returned to OpenBot

One sender only. The MCP process is the only thing that touches a LinkedIn control. OpenBot keeps the decision and the record.

Rules that make this single-sender rather than dual-sender:

- The existing linkedin.com activation deny stays exactly as it is. The Campaign Agent must never regain the ability to click Connect or Send through computer_click. Its direct LinkedIn browser access stays navigate-and-read.
- - Every write reaches LinkedIn only as an MCP tool call passing through the plugin gateway in server/src/plugins/tools.ts, which checks the grant, evaluates policy, and writes the audit row before calling out.
  - - The MCP's own sending tools are preserved, not removed. Deleting them would only push sending back into ungoverned clicking.
   
    - ## Call the MCP tools, or wrap them
   
    - Option A, call the existing tools through the authorization controller. Register the MCP server as a custom plugin, classify connect_with_person and send_message as writes, and gate them in the controller.
   
    - Option B, replace them with thin governed wrappers. Add a small module in this fork exposing governed_connect and governed_send, which require an authorization token minted by OpenBot and then delegate to the same underlying ConnectionActions and message sender.
   
    - Recommendation: Option A for the read-only pilot, Option B before any volume.
   
    - Reasoning. Option A needs no fork changes and stays trivially rebaseable against a repository that ships releases most weeks. But its authorization is advisory: the token lives in OpenBot, while the MCP's own gate is a boolean confirm_send argument supplied by whatever client calls it. Option B moves the gate inside the process that owns the browser, so a send cannot happen without a token OpenBot signed. That is the only version that survives an operator running uvx by hand.
   
    - Do not do both at once, and do not reimplement Connect discovery in OpenBot. That is the dual-sender failure mode.
   
    - ## Controls to preserve, and their real status
   
    - The brief asks that OpenBot's review card, Start authorization, recipient and send limits, attempt ledger, and unknown-outcome handling be preserved. Most of these have to be built first.
   
    - Review card. Not present. OpenBot has a components gallery and a generative UI path to build it on, but the card itself is new work.
   
    - Explicit Start authorization. Not present. No approval token exists anywhere in the repository. This is the most important gap, and tools/deploy/README.md already names it.
   
    - Recipient and send limits. Not present for LinkedIn. The recipient helpers under app/src/components/channels govern which coworker a channel addresses, not outreach targets.
   
    - Attempt ledger. Partially available. server/src/work/queue.ts and the audit trail provide durable rows, leases, and an attempt cap, but they count work items and tool calls rather than invitations per prospect. A per-prospect ledger keyed on profile URL is new work.
   
    - Unknown-outcome handling. Present in the MCP, absent in OpenBot. connect_with_person and send_message already return outcome_unknown with retry_safe false and omit the sent field. OpenBot must store that verbatim and refuse an automatic retry, because a repeat can invite or message twice.
   
    - Concurrency is already sound on the MCP side. SequentialToolExecutionMiddleware serializes calls with an in-process lock plus a cross-process profile lease, so two clients cannot drive one Chromium profile at once. OpenBot should not add a second queue in front of it.
   
    - ## Capability comparison, step by step
   
    - Search. MCP is better. search_people builds a validated URL with location, connection-degree, and currentCompany facets and refuses a filter LinkedIn would silently ignore. OpenBot would drive the same search by generic clicking and typing.
   
    - Profile extraction. MCP is better. get_person_profile returns named sections with pagination control over experience, education, skills, certifications, and posts. OpenBot reads whatever the page happens to render.
   
    - Drafting. OpenBot is better and should keep it. Drafting is model and policy work grounded in campaign context, evidence, and exclusions. The MCP has no drafting concept at all; note is just a string argument.
   
    - Connect discovery. MCP is far better, and this is the widest gap. linkedin_mcp_server/scraping/connection_actions.py classifies relationship state from URL patterns and ARIA attribute presence only, never label text, so a German or relabelled page classifies identically. It opens the invite through the vanityName custom-invite deeplink and falls back to the More menu. A generic clicker keyed on a visible label is precisely what breaks when LinkedIn renames a button.
   
    - Note insertion. MCP is better. It handles both invite dialog layouts currently in the wild, distinguishes the persistent Premium nudge banner from a real quota block, and reports note_sent as delivery rather than as textarea fill.
   
    - Send. MCP is better mechanically, OpenBot is better on authority. Use the MCP for the click and OpenBot for the permission. This split is the whole integration.
   
    - Result verification. MCP is better. It re-reads the action area after the write and separates sent, not sent, and unknown, with retry_safe marking the point after which a retry can duplicate. OpenBot currently depends on a human confirming visually during takeover.
   
    - Net: the MCP owns the hands for search, extraction, Connect discovery, note insertion, submit, and verification. OpenBot owns the brain and the record for campaign scoping, drafting, authorization, limits, ledger, and audit.
   
    - ## Risks
   
    - Licensing. Apache 2.0 upstream, MIT for OpenBot. Compatible for combination. Apache 2.0 obliges us to retain LICENSE and NOTICE, state our changes, and not use the project's marks to endorse a derivative. Keep this fork's NOTICE intact and record modifications.
   
    - LinkedIn account. The dominant risk, and not a licensing question. LinkedIn's User Agreement prohibits automated access; the upstream README says so and offers no warranty of account safety. Restriction or permanent ban is a realistic outcome. Use a single owned account with explicit written owner consent, never a customer's or an employee's account without it.
   
    - No rate limiting. The MCP ships no send caps by design and its FAQ puts volume squarely on the operator. Limits must come from OpenBot. Until they exist there is no automated campaign, only supervised single actions.
   
    - Security. The MCP holds a live logged-in LinkedIn session as a browser profile on disk under the linkedin-mcp directory, with portable cookies exported to cookies.json. That file is a bearer credential for the account: it is enough to act as the user without a password or MFA. Keep it off shared volumes, out of container images and backups, and never in this repository. The MCP also opens an interactive login window and runs a local daemon with a profile lease; bind it to loopback and treat it as sensitive as the OpenBot gateway.
   ## Test 1, read only

  Purpose: prove search and extraction quality with zero write risk.

  Scope: up to five Chicago data-governance leaders.

  Steps: call search_people with keywords covering data governance leadership, location Chicago. Then call get_person_profile for each result, at most five.

  Return per person: name, profile URL, headline, company, location, and the specific evidence supporting data-governance relevance, taken from the profile rather than inferred.

  Constraints. No connection request, no note, no message. Only tools annotated readOnlyHint are granted: search_people, get_person_profile, get_company_profile. connect_with_person and send_message are not granted for this test. The linkedin.com activation deny in the OpenBot policy stays in force throughout. Note that get_conversation and search_conversations are not read-only despite reading, because enumerating inbox rows selects them and can mark messages read; both are out of scope.

  Pass criteria: five plausible people with working profile URLs and real cited evidence, no fabricated facts, no write tool invoked, and one complete audit row per call.

  ## Test 2, one person, owner approved

  Purpose: compare the MCP's full connect, add note, send, verify cycle against OpenBot's current human-takeover path, once, on one recipient.

  Preconditions, all required. The LinkedIn account owner approves in writing for this one named recipient. The recipient is a legitimate business contact. A person is present for the whole run. The send cap is one. The recipient is written to the ledger before anything is attempted.

  Steps. OpenBot prepares the prospect and the drafted note and shows the review card. A human presses Start, which mints a single-use authorization scoped to one recipient. The authorized call reaches connect_with_person with the note. The returned status, note_sent, and retry_safe are written to the ledger verbatim. OpenBot then independently re-reads the profile to confirm the observed state rather than trusting the return value alone.

  What is measured. Whether Connect was found without label matching. Whether the note actually attached rather than merely filling the textarea. Whether a Premium quota block is reported as custom_note_limit_reached instead of a generic failure. Whether the returned status matches the independent re-read. How an unknown outcome is handled.

  Stop conditions. Any outcome_unknown ends the test with no retry and a human check of the profile. A custom_note_limit_reached result is a pass for reporting and a stop for sending. Nothing proceeds to a second recipient regardless of result.

  Explicitly out of scope: send_message. Profile-targeted send_message can open a separate DM instead of replying in an existing thread, per upstream issue 483, so it does not belong in a first controlled test.

  ## Not done here, on purpose

  No OpenBot pull request. No merge. No deployment. No LinkedIn login or action. No removal of the MCP's sending capabilities. No change to the upstream repository.

  Next decision for the owner: approve Test 1, and choose Option A or Option B before anything writes.
  
    - Deployment. Chromium via Patchright is heavy, and the hosted OpenBot deployment already omits per-coworker computers because no container platform grants a Docker socket, so every coworker shares one browser and therefore one LinkedIn identity. Adding the MCP means a second Chromium in the image; size for it. The recommended uvx latest configuration auto-updates on every start, which keeps selectors working but introduces unreviewed code into a governed deployment. Pin a version and update deliberately.
   
    - Maintenance. Upstream is very active, with well over a thousand commits and frequent fixes tracking LinkedIn DOM changes. That activity is a feature, because selector rot is the main failure mode, but it makes a heavily patched fork expensive to carry. Prefer a thin wrapper plus frequent rebase over edits spread through the scraping modules.
   
    - Proxy guidance. The upstream README recommends residential proxies and carries paid sponsor codes. Treat that as vendor placement, not architecture. Signing in from an unusual address is itself a risk signal; prefer the account's usual network.
   
    - Shared-identity audit gap. Because one browser serves every coworker, the audit trail proves which coworker called a tool but not which human was behind an account-level LinkedIn action. The per-prospect ledger should record the authorizing person explicitly.
   
    - 
