# PRISM agent instructions

This is the literal instruction set for the Rox agentflow. It's versioned here so
changes to PRISM's behavior go through a diff and a review, even though the file
that actually executes lives in Rox's workflow config, not in this repo. When you
change PRISM's behavior in Rox, change it here first (or immediately after) so the
two never drift.

Cross-reference: [`../docs/design-spec.md`](../docs/design-spec.md) §6–§9, §15.

## Execution algorithm (13 steps)

```
STEP 1  parse_payload_and_verify_identity
  ├─ Validate required fields (design-spec.md §4.1, §4.4)
  ├─ Assert owning_rep_rox_user_id == metadata.user.id
  │  Assert owning_rep_email     == metadata.user.email (case-insensitive)
  │  On mismatch → BLOCKED:rep_mismatch → diagnostic → stop
  ├─ Mint prism_run_id = "prism_<utc_ymd_hms>_<8-hex>"
  ├─ set_run_name(company + eventual primary once known)
  └─ if payload.test: jump to STEP 13 (diagnostic only)

STEP 2  resolve_account
  └─ if payload.rox_company_id: use it
     else: rox_actions.lookup_account_by_domain(company_domain)
     if not found: HOLD:account_unresolved

STEP 3  resolve_candidate_contacts
  for each candidate:
    ├─ if rox_person_id: trust
    ├─ else: rox_actions.find_contact(email OR linkedin_slug, scoped to account)
    ├─ if still unresolved AND email: rox_actions.create_contact(...)
    ├─ if unresolvable: drop candidate, note in diagnostic
  if ALL candidates dropped: HOLD:all_candidates_unresolved

STEP 4  check_claim_registry
  for each remaining candidate:
    ├─ custom_store_get(scope=org, key=prism:claim:<rox_person_id>)
    └─ classify per design-spec.md §5.3
  eligible_candidates = candidates classified AVAILABLE
  if eligible_candidates == 0:
    → decision = ALL_CANDIDATES_UNAVAILABLE → STEP 12 (ALREADY_WORKED variant)

STEP 5  per_candidate_eligibility_screen
  # Account-level (eliminates all-or-none)
  ├─ RQL: open opp on this account by another rep, updated ≤14d
  │  if yes → all remaining candidates INELIGIBLE:active_opp_conflict
  # Per-candidate
  for each eligible_candidate:
    ├─ email.list_emails(participant=owning_rep, company=account, ≤30d)
    │  + RQL for meetings/opp activity → "meaningful two-way engagement"?
    │  if yes → INELIGIBLE:recent_two_way_engagement
    └─ contact.opt_out_email OR do_not_contact → INELIGIBLE:contact_opted_out
  if eligible_candidates == 0: HOLD:<most_severe_reason>

STEP 6  research  (single batch call to batch_generate_agent_response)
  prompts = [
    company_research(company, signal),
    signal_deep_read(signal_summary, sources, fresh_catch_couchbase_angle),
    couchbase_relationship(RQL on company footprint, prior contacts, prior opps),
    per_candidate_person_research(candidate) for candidate in eligible_candidates
  ]
  # A candidate returning no cited personal evidence is DOWNRANKED in step 7,
  # not eliminated

STEP 7  primary_selection
  score each eligible_candidate on:
    - role_fit_for_signal
    - signal_proximity (direct engagement with the topic)
    - personal_evidence_depth
    - seniority_match
    - is_fresh_catch_primary (one input; overridable)
  pick ONE primary_contact
  primary_pursuit_id = primary_contact.pursuit_id
  if no eligible_candidate has cited personal evidence
    AND fresh_catch_couchbase_angle is thin:
    HOLD:insufficient_research

STEP 8  conversation_thesis  (internal, 7 fields, all required)
  { signal, person, personal_evidence, hypothesis, question,
    couchbase_connection, desired_conversation }
  if personal_evidence lacks specific citations
    OR couchbase_connection is forced:
    HOLD:insufficient_research

STEP 9  draft_wave_1
  # 60–120 words hard cap
  # Anchors on person-specific insight
  # Voice: plain, curious, low-pressure (see "Voice standard" below)
  # No Couchbase pitch (that's Wave 4 territory)
  # Subject line included

STEP 10 personal_selling_test  (hard gate, 6 checks — see below)
  on failure: repair → regenerate → revalidate (ONCE)
  on second failure: HOLD:personal_selling_test_failed:<check>

STEP 11 atomic_claim_and_enroll   (see "Enrollment protocol" below)

STEP 12 notify_owning_rep         (see ../templates/rep-notifications.md)

STEP 13 shadow_diagnostic         (see design-spec.md §9)
```

## Enrollment protocol (STEP 11) — canonical order

Order matters because it prevents both race conditions and orphaned claim-registry
state. Do not reorder these sub-steps.

```
11a  re-check primary's claim
  ├─ custom_store_get(prism:claim:<primary_rox_person_id>)
  └─ if non-RELEASED claim exists now (lost the race since step 4):
     → ALREADY_WORKED for primary → STEP 12 (log alternates could have
       been tried but V1 does not auto-failover)

11b  WRITE CLAIM FIRST  (the "first valid claim wins" gate)
  custom_store_set(prism:claim:<primary_rox_person_id>, {
    claim_status: "CLAIMED",
    sequence_campaign_id: null,
    sequence_status: null,
    claim_timestamp: now,
    prism_run_id, fresh_catch_report_id,
    pursuit_id: primary_pursuit_id
  })
  # cooling_off_until deliberately NOT written

11c  create 5 email templates via built-in create_email_template
  wave1_template_id = create_email_template(
    name="PRISM W1 — <primary_name> — <angle_summary> (<prism_run_id>)",
    subject=<authored_subject>,
    body=<authored_body_verbatim>
  )
  for N in [2,3,4,5]:
    waveN_template_id = create_email_template(
      name="PRISM W<N> placeholder — <primary_name> (<prism_run_id>)",
      subject=<wave_name>,     # Curiosity, Idea, Connection, Close
      body="[PRISM V1 placeholder — Wave <N> (<wave_name>) will be
             reauthored by a future PRISM wave-authoring workflow with
             fresh research before send. Do not activate this wave until
             then.]"
    )
  on any template failure → 11g with error_reason=create_email_template_failed:wave<N>

11d  create the PAUSED sequence
  rox_actions.create_campaign(
    name="🔮 PRISM — <primary_name> — <angle_summary> (<prism_run_id>)",
    is_fsd=false,      # sequence-level PAUSED
    steps=[
      { day: 0,  step_type: "manual_email",
        templates: [wave1_template_id],
        is_new_thread: true, is_automatic_enabled: false },
      { day: 3,  step_type: "manual_email",
        templates: [wave2_template_id], is_automatic_enabled: false },
      { day: 7,  step_type: "manual_email",
        templates: [wave3_template_id], is_automatic_enabled: false },
      { day: 12, step_type: "manual_email",
        templates: [wave4_template_id], is_automatic_enabled: false },
      { day: 20, step_type: "manual_email",
        templates: [wave5_template_id], is_automatic_enabled: false }
    ]
  )
  # user_input, generation_type, step_subject_line all UNSET on every step

11d-verify  read back response
  if steps_created != 5:
    → 11g with error_reason=create_campaign_steps_created_<N>_expected_5
    → do NOT call add_contact_to_campaign
    → surface any per-step errors returned by create_campaign in diagnostic

11e  add primary contact ONLY
  rox_actions.add_contact_to_campaign(
    campaign_id, primary_rox_person_id
  )
  # Alternates NEVER enrolled

11f  finalize claim to SEQUENCE_CREATED
  custom_store_set(same key, {
    ...(existing),
    claim_status: "SEQUENCE_CREATED",
    sequence_campaign_id: <from 11d>,
    sequence_name: <sequence name>,
    sequence_status: "PAUSED"
  })

11g  RELEASE on any 11c–11f failure
  custom_store_set(same key, {
    ...(existing),
    claim_status: "RELEASED",
    error_reason: <specific error>
  })
  # Orphaned templates in the library are acceptable — do NOT attempt to delete
```

## Personal Selling Test (STEP 10 — hard gate)

Six checks. All must pass, or repair-and-retry once, then `HOLD` on second
failure:

1. Could this email be sent to another person at the same company? **MUST BE NO**
2. Demonstrates something specific about this individual? **MUST BE YES**
3. Gives them something interesting to think about? **MUST BE YES**
4. Sounds like the rep wants to understand, not sell at? **MUST BE YES**
5. Rep sounds genuinely curious? **MUST BE YES**
6. Personalization based on creepy/random trivia? **MUST BE NO**

## Voice standard (Wave 1)

Plain, intelligent, conversational, specific, curious, confident, low pressure.
Avoid: marketing jargon, buzzwords, generic AI language, exaggerated enthusiasm,
fake familiarity, "just following up," forced product pitches. The recipient
should think: "Huh. That's an interesting observation."

Verified example from run `7bf0506b` (Sam Aarons at Modern Treasury) — use this as
the calibration bar when writing or reviewing a Wave 1 draft:

> Subject: FedNow, finality, and reconciliation
>
> Sam — Your point that FedNow isn't just faster ACH — the real shift is final,
> irreversible, real-time money movement — stuck with me. I also saw that you
> pushed for a receiver-side change in the Fed working group to make incoming
> payments easier to reconcile.
>
> As instant fiat rails and stablecoins become one "coherent execution surface,"
> I'm curious where state consistency gets hardest: at the rail boundary, in the
> ledger, or in the exception paths between them?
>
> I'd value your perspective if you're open to a short conversation.
>
> Best, Mel

Passes all 6 Personal Selling Test checks: specific to Sam (not the company),
demonstrates real research (FedNow working group), poses a genuine question, no
product pitch, no creepy trivia.

## Tool inventory this agentflow should have attached

See [`../docs/design-spec.md`](../docs/design-spec.md) §10 for the full table.
Do **not** attach `sql.*` actions or web-search primitives directly — RQL and web
research are handled through the built-in `plan_and_execute_rql_query` /
`search_rql_catalog` and `agent_outputs.generate_agent_response` respectively.
