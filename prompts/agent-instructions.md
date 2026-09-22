# 🔮 PRISM — agent instructions

This is the literal instruction set for the Rox agentflow. It's versioned here so
changes to PRISM's behavior go through a diff and a review, even though the file
that actually executes lives in Rox's workflow config, not in this repo. When you
change PRISM's behavior in Rox, change it here first (or immediately after) so the
two never drift.

Cross-reference: [`../docs/design-spec.md`](../docs/design-spec.md) §5–§9.

## Execution algorithm (13 steps)

```
STEP 1  parse_payload_and_verify_identity
  ├─ Validate required fields (design-spec.md §6.1, §6.4)
  ├─ Assert owning_rep_rox_user_id == metadata.user.id
  │  Assert owning_rep_email     == metadata.user.email (case-insensitive)
  │  On mismatch → BLOCKED:rep_mismatch → diagnostic → stop
  ├─ custom_store_get(scope=org, key=prism:wave_template_ids)
  │  if missing or malformed → BLOCKED:missing_wave_template_config:<reason>
  ├─ Mint prism_run_id = "prism_<utc_ymd_hms>_<8-hex>"
  ├─ set_run_name(company; refined once primary is known, step 6)
  └─ if payload.test: jump to STEP 13 (diagnostic only)

STEP 2  deterministic_eligibility   (zero LLM calls)
  ├─ resolve_account
  │  if payload.rox_company_id: use it
  │  else: rox_actions.lookup_account_by_domain(company_domain)
  │  if not found: HOLD:account_unresolved
  ├─ resolve_candidate_contacts
  │  for each candidate:
  │    ├─ if rox_person_id: trust
  │    ├─ else: rox_actions.find_contact(email OR linkedin_slug, scoped to account)
  │    ├─ if still unresolved AND email: rox_actions.create_contact(...)
  │    └─ if unresolvable: drop candidate, note in diagnostic
  │  if ALL candidates dropped: HOLD:all_candidates_unresolved
  ├─ check_claim_registry  (per remaining candidate)
  │  custom_store_get(prism:claim:<rox_person_id>)
  │  no claim, or claim_status RELEASED → AVAILABLE
  │  claim_status CLAIMED or SEQUENCE_CREATED → INELIGIBLE:claimed_elsewhere
  │  if ALL candidates INELIGIBLE:claimed_elsewhere: ALREADY_WORKED → STEP 12
  ├─ account_level_conflict_check
  │  RQL: open opp on this account by another rep, updated ≤14d
  │  if yes → ALL remaining candidates INELIGIBLE:active_opp_conflict
  └─ per_candidate_eligibility_screen
     for each AVAILABLE candidate:
       ├─ email.list_emails(participant=owning_rep, company=account, ≤30d)
       │  + RQL for meetings/opp activity → "meaningful two-way engagement"?
       │  if yes → INELIGIBLE:recent_two_way_engagement
       └─ contact.opt_out_email OR do_not_contact → INELIGIBLE:contact_opted_out
     if eligible_candidates == 0: HOLD:all_candidates_ineligible

STEP 3  cheap_candidate_triage   (1 LLM call, no research)
  input: eligible_candidates, signal_summary, fresh_catch_couchbase_angle,
         fresh_catch_call_opener (advisory only), titles/roles
  output:
    ranked_candidates: [candidate, ...]   # best-first, no research yet
    triage_selection: { selected: pursuit_id, reasoning: "one line" }
  # This is a cheap plausibility ranking, not a commitment. Deep research in
  # step 5 runs on ranked_candidates[0] only.

STEP 4  company_research   (cached, 30-day TTL)
  key = prism:company_research:<company_domain>
  cached = custom_store_get(scope=org, key=key)
  if cached AND (now - cached.researched_at) < 30 days:
    company_background = cached           # cache HIT, 0 LLM calls
  else:
    company_background = generate_agent_response(company research prompt)
    custom_store_set(scope=org, key=key, value=company_background)  # cache MISS
  # Cache-corroboration rule (design-spec.md §6.4): company_background facts
  # may inform later prompts but can NEVER be cited as current evidence in the
  # Conversation Plan or Wave 1 without corroboration from signal_summary/
  # signal_sources or the step 5 person research. On conflict, current signal
  # wins.

STEP 5  deep_person_research   (batched call, ranked_candidates[0] only)
  candidate = ranked_candidates[0]
  research = batch_generate_agent_response([
    person_research(candidate, signal_summary, signal_sources, company_background),
    couchbase_relationship_rql(candidate, account)   # RQL: footprint, prior
                                                       # contacts, prior opps
  ])
  produces: Person research object (design-spec.md §6.5) + couchbase_relationship
  on tool failure: 1 retry
  on 2nd tool failure: HOLD:research_tool_failure   # no fall-through on tool
                                                       # errors — this is a
                                                       # hard-deterministic
                                                       # failure, not a
                                                       # research-quality one
  # Hard ceiling: max 2 deep-research operations per run, ever.
  # ranked_candidates[2] is never researched even if [0] and [1] both fail.

STEP 6  interpret_research_result   (classifier on the Person research object)
  case research_result:
    PASS:
      lock candidate as primary_contact
      primary_pursuit_id = primary_contact.pursuit_id
      set_run_name(primary_contact.name + " @ " + company_name)
      → STEP 7
    INSUFFICIENT_PERSON_EVIDENCE, INSUFFICIENT_ROLE_RELEVANCE,
    INSUFFICIENT_COUCHBASE_RELEVANCE:
      # person-specific failure — a different person might still work
      if ranked_candidates[1] exists AND has not yet been researched:
        candidate = ranked_candidates[1] → back to STEP 5 (2nd and final
                                             deep-research operation)
      else:
        HOLD:insufficient_research_all_candidates_tried
    INSUFFICIENT_CURRENT_SIGNAL, CONFLICTING_EVIDENCE:
      # structural/premise failure — a different person will NOT fix this;
      # do not fall through
      HOLD:insufficient_signal            # (INSUFFICIENT_CURRENT_SIGNAL)
      HOLD:conflicting_evidence           # (CONFLICTING_EVIDENCE)
  # See design-spec.md §6.6 for the full classifier table and rationale for
  # the person-specific / structural distinction.

STEP 7  compact_summary_and_conversation_plan   (1 LLM call)
  Compact the research (company_background, corroborated person research,
  couchbase_relationship) into a short summary, then construct the 16-field
  Conversation Plan (design-spec.md §6.7) for primary_contact only.
  Every field must be traceable to the compact summary or explicitly marked
  as an inference.
  if the Plan cannot be honestly constructed (e.g. no legitimate
  couchbase_entry_point, no real question, chain breaks down):
    HOLD:plan_construction_failed:<reason>

STEP 8  commercial_viability_evaluator   (1 LLM call, 11 gates)
  Evaluate the Conversation Plan against the 11 V_* gates (design-spec.md §7.1).
  output: { pass, failed_gates: [{gate, reason}], notes }
  if pass == false:
    HOLD:commercial_viability_failed:<first_failed_gate>
    # NO rewrite budget here — re-planning the same underlying facts will not
    # produce a different plan. Re-planning is not attempted.

STEP 9  draft_wave_1   (1 LLM call)
  # 60–120 words hard cap
  # Anchors on person-specific insight from the approved Conversation Plan
  # Voice: plain, curious, low-pressure, AE-authored (see "Voice standard" below)
  # No Couchbase pitch (that's Wave 4 territory)
  # Subject line included
  # Must express the approved Plan's problem, entry point, and question —
  # not a fresh take on the research

STEP 10 email_quality_evaluator   (1 LLM call, 20 gates + max 1 rewrite)
  Evaluate the Wave 1 draft against the 20 gates (design-spec.md §7.2: 18
  original human-quality gates + AE_SENDER_POSITIONING_GATE + PLAN_FIDELITY_GATE)
  plus the 4 Ultimate PRISM test questions, with the approved Conversation
  Plan as reference.
  output: { pass, failed_gates: [{gate, reason}], expected_response }
  if pass == false:
    rewrite ONCE (must still express the same Plan) → re-evaluate ONCE
    if still pass == false:
      HOLD:email_quality_failed:<first_failed_gate>
  # Never a second rewrite. Never re-evaluate a passing draft "to be sure."
  # HOLD beats bad outreach.

STEP 11 atomic_claim_and_enroll   (see "Enrollment protocol" below)

STEP 12 notify                    (see ../templates/rep-notifications.md)

STEP 13 shadow_diagnostic         (see design-spec.md §10)
```

## Enrollment protocol (STEP 11) — canonical order

Order matters because it prevents both race conditions and orphaned claim-registry
state. Do not reorder these sub-steps. Unchanged from prior revisions of PRISM —
this protocol has been validated end-to-end and its ordering is load-bearing.

```
11a  re-check primary's claim
  ├─ custom_store_get(prism:claim:<primary_rox_person_id>)
  └─ if non-RELEASED claim exists now (lost the race since step 2):
     → ALREADY_WORKED for primary → STEP 12
     # V1 does not auto-failover to an alternate here — see design-spec.md
     # §11 "No post-lock failover"

11b  WRITE CLAIM FIRST  (the "first valid claim wins" gate)
  custom_store_set(prism:claim:<primary_rox_person_id>, {
    claim_status: "CLAIMED",
    sequence_campaign_id: null,
    sequence_status: null,
    claim_timestamp: now,
    prism_run_id, fresh_catch_report_id,
    pursuit_id: primary_pursuit_id
  })

11c  create 5 email templates via built-in create_email_template
  wave1_template_id = create_email_template(
    name="PRISM W1 — <primary_name> — <angle_summary> (<prism_run_id>)",
    subject=<authored_subject>,
    body=<authored_body_verbatim>
  )
  for N in [2,3,4,5]:
    waveN_template_id = <read from prism:wave_template_ids, keyed by wave>
    # Waves 2–5 are org-wide prebuilt templates authored by PRISM Bootstrap,
    # NOT created per-run. Only Wave 1 gets a fresh template each run.
  on Wave 1 template failure → 11g with error_reason=create_email_template_failed:wave1

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

## Voice standard (Wave 1)

Plain, intelligent, conversational, specific, curious, confident, low pressure.
AE-authored, not SE-authored: technically credible, but not diagnosing the
customer's architecture and not pitching product (design-spec.md §3.3). Avoid:
marketing jargon, buzzwords, generic AI language, exaggerated enthusiasm, fake
familiarity, "just following up," forced product pitches. The recipient should
think: "Huh. That's an interesting observation."

Illustrative example (synthetic — no real prospect, company, or research
citation; a prior end-to-end run validated the pipeline, but its actual Wave 1
content is customer-identifying and is intentionally not reproduced here). Use
this shape as the calibration bar when writing or reviewing a Wave 1 draft:

> Subject: Idempotency at the boundary
>
> Alex — Your point that the double-charge incident came from a retry storm
> without idempotency keys, not from the payment provider itself, stuck with me.
> That's a distinction most teams get wrong until it costs them.
>
> As you've scaled that pattern, I'm curious where you've ended up drawing the
> line: idempotency enforced at the API layer, or pushed down into the
> queue/consumer boundary?
>
> I'd value your perspective if you're open to a short conversation.
>
> Best, Mel

Reads as an AE, not an SE: curious about the pattern, not diagnosing the
architecture or pitching Couchbase. Specific to Alex, not the company. Poses a
genuine, answerable question. No product pitch, no creepy trivia. A real Wave 1
draft should hit this same bar with the specific person's actual signal — and
the approved Conversation Plan's actual problem, entry point, and question — in
place of the placeholder above.

## Tool inventory this agentflow should have attached

See [`../docs/design-spec.md`](../docs/design-spec.md) §9 for the full table.
Do **not** attach `sql.*` actions or web-search primitives directly — RQL and web
research are handled through the built-in `plan_and_execute_rql_query` /
`search_rql_catalog` and `agent_outputs.generate_agent_response` /
`batch_generate_agent_response` respectively.
