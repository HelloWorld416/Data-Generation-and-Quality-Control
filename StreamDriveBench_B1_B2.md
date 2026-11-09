# B. Data Generation and Quality Control

This appendix documents the data-generation and quality-control procedure used to build the current StreamDriveBench release. The implementation is an event-grounded pipeline: it first discovers and confirms observable driving transitions, then writes questions about those confirmed transitions, and finally accepts a question only after independent temporal, causal, and event-association checks.

The description below follows the Boreas v2 construction flow in `qa_gen_pipeline_boreas_v2_repair`. The released Boreas pool contains 1,500 automatically verified questions from 125 videos: 580 History QA (HGI), 530 Prediction QA (AD), and 390 Event QA (ETP). The generation model is `gpt-6.1-sol` with low reasoning effort. Automatic verification is recorded separately from human approval; a question marked `human_approved=false` is still pending human review.

## B.1 Event Discovery and QA Generation

### Input normalization and temporal units

Videos are recursively inventoried and decoded from their first valid presentation timestamp. The pipeline samples a one-frame-per-second grid and preserves the original aspect ratio by letterboxing to 910 x 512 pixels. Missing timestamps, non-monotonic timestamps, stale frames, and decode gaps are treated as unavailable evidence rather than filled by interpolation. All temporal fields in the prompts and release files are integer seconds on this grid.

Each video is partitioned into non-overlapping discovery windows `[start, end_exclusive)` of at most 30 seconds. A window may be shorter at the end of a video, but it must contain enough valid observations for a boundary to be confirmed. The current Boreas build allows at most one confirmed event per window. This is a sampling and quota policy, not a claim that every event in a video is discovered.

### Stage 1: Event Discovery Prompt

The discovery request sees only the images in one ownership window. It proposes events but does not generate questions or answers.

```text
Discover driving events ONLY; do not generate questions or answers. Images cover exactly one ownership window [start,end_exclusive), without neighboring context. Select at most max_events distinct salient events with a visible atomic transition and before/after evidence INSIDE the window. Prefer interactions, stop/start, crossing, turning and signal changes. Use absolute integer seconds. No hidden intent, metric speed or inferred telemetry. Do not duplicate the same actor's same atomic transition or relabel an ongoing state. Rank by clarity and return no events if unsupported. Candidates require independent boundary confirmation.
```

The structured response contains, for each candidate, an actor, behavior, before-state, after-state, an approximate transition second, evidence seconds, and uncertainty notes. An empty event list is valid when no clear transition is present.

### Stage 2: Independent event-boundary confirmation

Every proposed event is rechecked with a dense set of window images using a separate verifier. The verifier must identify the **first observable** second at which the atomic transition is satisfied and the immediately preceding second at which it is absent. Both adjacent seconds must be supplied and visually clear. It rewrites the actor and state descriptions using only supported visual facts, rejects ambiguous identity or visibility, and never claims sub-second precision or hidden intent. Events that fail this check are discarded before QA authoring.

The resulting event record stores the confirmed transition, `last_absent = first_observable - 1`, evidence frames, discovery-window bounds, and the source episode identifier. The event identifier is immutable and is used to bind all questions and private labels to the same transition.

### Stage 3: Shared QA-authoring prompt

After all candidates in a window have passed boundary confirmation, one authoring request receives the confirmed events and the shared window images. It must return exactly one question set for each confirmed event. For each requested family it returns one question or an explicit evidence-based omission reason; it cannot silently omit a family or invent a question to fill a quota.

The common authoring instructions are:

```text
Generate driving QA ONLY for the supplied confirmed_events. Their boundaries have already passed independent verification. Do not invent events, change their identifiers or boundaries, or attach questions to a rejected event. All events share the supplied window images: return exactly one question_set per confirmed event_id.

For EACH event actively attempt one question for EACH requested_families entry (default HGI, AD, ETP) concerning the same identifiable actor and continuous behavior episode. Never silently omit a family: provide either exactly one question or an explicit evidence-based omission reason for each requested family. Never generate a family absent from requested_families. Do not invent unsupported questions to fill quotas. Do not output answers. Avoid duplicate model inputs across events.

Every question specifies history_start_second. Select only the history needed for the question, from ownership_start through query_second, with query_second-history_start_second <= 30. A full past 30 seconds is not required. HGI needs an earlier observation; AD and ETP may use zero history when sufficient. Questions must use a currently identifiable target, avoid future attributes and event labels, and cite only observations available at query_second in rationale_from_prefix.

The author sees the complete discovery window and the confirmed event. This is offline, outcome-aware authorship, not a causal prediction. Subsequent independent audits must establish causal validity and answer support.
```

The author is also instructed to use short, natural passenger-style English, parallel mutually exclusive choices, no frame numbers or exact timestamps in the question, no code-like answer options (except `SILENT`), and no long coordinate-like target descriptions. Naturalness is part of validity.

### History QA (HGI) Generation Prompt

The HGI branch asks about a **current** state or identity whose answer genuinely depends on visual history. It has 2–4 plausible choices and no `SILENT` option.

```text
HGI: Ask about a CURRENT state or identity that genuinely needs historical visual evidence, not a fact available from the latest frame and not an arbitrary time lookup. Use the shortest useful history within the discovery window (at most 30 seconds before query_second). Include at least one earlier observation. Give 2–4 plausible, mutually exclusive choices and do not presuppose the answer or a future event. Use a concise current target description. Do not expose timestamps or benchmark terminology in the question.
```

Typical form: `How did the cyclist now ahead get into our lane?` The author may use a short history, but the subsequent audit must show that the current frame alone is insufficient and that the selected historical frames are necessary.

### Prediction QA (AD) Generation Prompt

The AD branch predicts an observable action or change in the interval immediately after the query. The current protocol uses a three-second horizon with interval semantics: `(query_second, query_second + 3]`. It does not ask for the state at one exact endpoint frame.

```text
AD: Ask what the designated target will visibly do in the next few seconds, with horizon_seconds=3 and evaluation interval (query_second, query_second+3]. The query must precede the confirmed transition and the full interval must lie inside the discovery window. Give 2–4 plausible alternative actions or changes, with no SILENT option and no assertion of the actual future outcome. If multiple alternatives occur in the interval, ask for the FIRST action/change or omit the question. Do not turn the task into an exact future-frame state lookup.
```

Typical form: `What will the van ahead do in the next few seconds?` The release stores `prediction_horizon_seconds=3`, `prediction_semantics=action_within_interval`, and private `future_start_second`/`future_end_second` fields.

### Event QA (ETP) Generation Prompt

The ETP branch creates a conditional online-monitoring question. The model must remain silent until one new, observable, atomic driving-state condition becomes true, and then select an answer about that condition. `SILENT` is always one option and is placed last during release balancing.

```text
ETP: Ask the model to stay silent until ONE observable, atomic driving-state condition becomes true for the currently identifiable target, then answer a concrete discriminative question. The condition must be false at injection and plausibly may or may not occur. Use 2–4 non-SILENT choices plus exactly one SILENT choice. Non-SILENT choices describe possible answers when the condition triggers; they must not encode timeout, non-occurrence, lost visibility, or a hard-coded output token. Do not use compound and/or predicates, camera exit, vague “soon”, or hidden physical attributes. Prefer a visible stop, start, turn, lane change, crossing, or signal transition.

Choose monitor_duration_seconds once before independent monitoring. It must be an event-specific complete interval of 1–60 seconds that contains the confirmed transition and any necessary post-transition context. Explain the choice in monitor_duration_reason. Do not shorten the interval after verification to manufacture a negative example, and omit the question when no meaningful complete interval exists.
```

Typical form: `When the van ahead starts turning, which way does it turn? Stay silent until then.` A conditional “when/until” is not a claim that the event will happen. A negative ETP means only that the condition was absent throughout its explicitly scheduled interval; it does not mean that the event will never happen.

### Preflight, quota scheduling, and release construction

Before any expensive verification request, deterministic checks validate query placement, history bounds, choice counts, horizon fields, ETP duration fields, timestamp leakage, coded options, hard-coded answer emission, and timeout options. The quota scheduler reserves only still-missing family slots before submitting a window, so rejected questions release their reservation and do not lower the acceptance standard. It stops when all family targets are met or when the candidate-window/API budget is exhausted.

For the Boreas build, 125 videos were sampled with a fixed seed; windows were processed in rotating batches, with at most one window per video in a batch and at most one event per window. Each selected video is decoded to its 1-Hz frame store on first use. The build target was HGI=580, AD=530, ETP=390 (1,500 total), with up to 24,000 API attempts and up to 1,740 assigned windows. Every completed window is checkpointed, successful API responses are cached, and the build plan freezes the input hashes, model, sampling seed, limits, and source-code hashes.

The release separates public inputs from evaluator-only labels:

* `model_inputs.json` contains the question, choices, target description, predicate, query time, and input policy.
* `labels.json` and the evaluator-only portion of `queries_all.json` contain answers, evidence frames, trigger times, and event links.
* `events.json` and `event_groups.json` preserve the confirmed transitions and split-group constraints.
* `option_mapping_audit.json` records the global answer-position balancing; ETP keeps `SILENT` in the final position.

The implementation references for this appendix are `SPECIFIC_BUILD.md`, `vendor/qa_gen_pipeline_boreas_v2_repair/src/streamdrivebench/protocol.py` (window and authoring contracts), `checks.py` (schemas and semantic checks), `pipeline.py` (verification order and causal payload guards), and `specific.py` (quota scheduling and publication).

## B.2 Automatic QA Verification

Verification uses separate requests and schemas from authoring. Verifiers are not shown an author-provided answer when determining the label. A candidate is accepted only if all required checks pass; otherwise its rejection reason is retained in the window checkpoint and summary statistics.

### Verification criteria by QA type

| QA type | Evidence shown to the verifier | Required checks | Positive/negative interpretation | Representative rejection cases |
| --- | --- | --- | --- | --- |
| **History QA (HGI)** | Current-only ablation: frame at `t`; full mode: selected causal history ending at `t` | Target is identifiable; choices are unambiguous; answer is supported by supplied evidence; no future presupposition; full-history mode reports `history_necessary=true`; current-only mode must not already solve the question | The private label is an exact choice supported by historical evidence, including at least one frame before `t` | Latest frame already answers the question; no earlier evidence; ambiguous target/choices; future or answer-revealing wording |
| **Prediction QA (AD)** | Causal prefix through `t` for question validity, then independent future interval `(t,t+3]` for labeling | Prefix audit confirms target, wording, and meaningful alternatives without guessing the future; future audit judges the whole interval, not only its endpoint; the first action/change is used when the question says “first” | The private label is the observed choice whose action/change occurs in `(t,t+3]`; transient actions count | Future outcome leaked in the question; target or options ambiguous; endpoint-only state lookup; multiple alternatives occur without an explicit first-action rule; no supported future evidence |
| **Event QA (ETP)** | Causal prefix through injection `t`; then every 1-Hz observation through the preselected deadline `t+monitor_duration_seconds` | Injection audit confirms a currently identifiable target, one atomic condition, condition false at injection, no future presupposition, and natural wording; monitor audit checks visibility and duration suitability and finds the first observable trigger | **Positive:** before the first trigger answer is `SILENT`; at the first clear trigger the exact non-SILENT choice is emitted. **Negative:** if the condition remains absent for the entire scheduled interval, the label is `SILENT` | Condition already true at injection; compound condition; target disappears or is unresolved; interval too short or cut off; timeout/lost-visibility answer option; backdated or later-than-first trigger |

### HGI verification

HGI is deliberately tested with a current-frame ablation. The current-only verifier may answer only when the single frame is sufficient. If it can answer unambiguously, the candidate is rejected as a non-history question. The full-history verifier must independently support the exact answer, identify the target, mark the choices unambiguous, and cite evidence that includes an earlier observation. This implements the distinction between “the answer is visible now” and “the answer requires reconstructing what happened before now.”

### AD verification

AD has two independent passes. The prefix pass sees no future frames and therefore checks only causal validity: target identity, natural wording, meaningful alternatives, and absence of future leakage. The future-label pass sees the complete three-second interval `(t,t+3]` and assigns the answer from visible action/change. It rejects uncertainty, treats any transient action in the interval as relevant, and rejects a candidate when multiple alternatives occur unless the question unambiguously asks for the first one. The current frame at `t` is excluded from the forecast interval.

### ETP injection and monitoring verification

ETP verification is split at the injection boundary. The injection verifier sees only the causal prefix and checks that the condition is a new, currently unmet, single atomic driving-state change. Conditional wording such as “when it happens” is allowed and does not count as future presupposition. Passing a landmark, clearing an occluder, or leaving the camera view is not a driving-state change.

The monitor verifier then scans every scheduled second without seeing the proposed answer or trigger. It must return the first observable satisfaction and the preceding absent second. For a positive, a separate commitment audit checks the two causal cutoffs: at `trigger-1` the model must still answer `SILENT`, while at `trigger` it must be able to select the labeled non-SILENT option. For a negative, `first_observable=-1`, `last_absent=deadline`, and `event_absent_throughout=true` are valid only when the target remains sufficiently visible and the chosen interval is suitable. Unknown visibility is invalid evidence, not a negative label.

### Cross-checking and limitations

An association verifier independently cross-checks the answer, option coverage, target identity, continuous behavior episode, and same atomic transition. It rejects questions about an unrelated property of the same vehicle, missing correct choices, inaccurate direction/motion labels, or semantic duplicates. Strict triplets are exported only when HGI, AD, and ETP all link to the same atomic transition.

All checks are automated model-assisted checks and preserve evidence, schemas, and rejection reasons for audit. They do not replace human ground truth. Authoring observes the complete discovery window and confirmed event, so offline outcome-aware selection and wording bias cannot be eliminated by the later causal audits. Window sampling and the one-event-per-window cap also mean that the released events are a verified subset rather than an exhaustive event inventory.

