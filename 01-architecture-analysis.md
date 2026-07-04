# Part 1 — Architectural Analysis of the Source Prompt

Subject: the Claude Fable 5 claude.ai consumer system prompt (~1,600 lines, ~42k tokens).
Method: treat it as a production codebase — map its modules, extract the design
principles that make it work, and identify where structure follows accident rather
than intent.

## 1.1 Module inventory and token budget

The prompt decomposes into five de-facto layers, though nothing in the text declares
them:

| Layer | Contents | Approx. share |
|-------|----------|---------------|
| A. Identity & product knowledge | product_information, identity preamble, model strings, feature list | ~4% |
| B. Behavioral policy | refusal_handling, child safety, legal/financial, tone_and_formatting, user_wellbeing, evenhandedness, mistakes/criticism, knowledge_cutoff | ~15% |
| C. Capability modules | memory stub, artifact storage API, MCP app suggestion flow, computer_use + skills, search_instructions, image search, Claudeception, citations | ~33% |
| D. Tool contracts | 13 inline JSON schemas with long prose descriptions | ~40% |
| E. Environment config | user location, available_skills registry, network allowlist, filesystem mounts | ~8% |

Two observations follow immediately. First, the single largest cost center is tool
schemas that are always loaded whether or not the conversation touches them (a
recipe widget schema is resident during a tax question). Second, behavioral policy —
the part that most needs the model's attention — is a minority of the token budget
and is interleaved with configuration rather than isolated.

## 1.2 Core reasoning model

**Decision procedures over principles.** The prompt's most distinctive choice is
that it almost never asks the model to derive behavior from values. Instead it
compiles values down to trigger→action tables with worked examples: "Is Mark Walter
still the chairman of the Dodgers?" → one search; "who is Dario Amodei" → no search;
"what has Dario Amodei done lately" → search. This is a deliberate reliability
trade: enumerated cases generalize worse than principles but fire far more
consistently. The prompt consistently pays the token cost to buy that consistency.

**Uncertainty management via information volatility.** The search policy is really
a freshness classifier: rate every query by how fast its ground truth changes
(timeless → never search; positional/status → always search; breaking → search
immediately) plus a confabulation firewall — the "unrecognized entity rule" — that
converts a specific internal signal ("I can't place this capitalized word") into a
mandatory action (search before answering). Uncertainty is managed behaviorally,
not by asking the model to report calibrated confidence.

**Answer construction.** Formatting is treated as a budgeted resource: prose by
default, bullets only when structurally necessary, format-per-surface rules
(conversational answers must not look like reports), and per-genre file-vs-inline
routing. The underlying principle: output shape should be chosen by the content's
consumer, not by the model's convenience.

**Planning and self-correction are vestigial.** Planning appears exactly once
("for complex queries, first make a research plan") and only for search.
Self-checking appears exactly once (the copyright pre-response checklist). There is
no general plan→verify loop, no error-recovery doctrine, no notion of iterating on
a draft. For a chat product this is tolerable; for agentic work it is the largest
capability gap.

## 1.3 Tool usage architecture

**Routing hierarchy.** A three-tier priority is explicit: internal/personal tools
first, web second, combined for comparative queries — with linguistic cues ("our",
"my") as the classifier. Connector acquisition is a pipeline with an opt-in gate:
`search_mcp_registry` → `suggest_connectors` → user choice → call, and third-party
consumer tools may never be invoked without that gate even when already connected.
The gate encodes a value (the user picks the vendor, not the model) as a mechanical
workflow, which is exactly how values survive contact with an eager model.

**Compute budgets.** Tool effort is explicitly tiered: 1 call for single facts,
3–5 medium, 5–10 deep, and a hand-off threshold ("if it would take 20+ calls,
suggest the Research feature"). This is a scheduler: it prevents both under-research
(confabulation) and over-research (latency/cost), and it defines an escalation path
out of the current execution mode.

**Progressive disclosure via skills.** The most architecturally important idea in
the file: the prompt does not contain document-production knowledge; it contains a
*router* to SKILL.md files and an unconditional rule to read them before acting.
The system prompt is a kernel; skills are loadable drivers. This is what keeps the
prompt maintainable at all — domain knowledge is versioned outside it.

**Environment contracts.** File locations form a memory hierarchy with visibility
semantics: uploads (read-only input), `/home/claude` (invisible scratch),
`/mnt/user-data/outputs` (the only user-visible surface) — plus the rule that work
not copied to outputs and presented effectively does not exist. Network access is an
allowlist with a structured error channel (`x-deny-reason`). These are stated as
invariants, not advice, which is correct: the model cannot negotiate with a mount
table.

## 1.4 Agent architecture

Thin by design. Execution is single-pass: classify the query, route to tools,
respond. There is no verification stage, no recovery framework beyond tool-local
hints (auth failure → re-suggest connector), no state carried across turns
(memory is present in the schema but disabled), and no task decomposition beyond
search planning. The prompt's agency model is "one competent turn," which is the
correct scope for chat and the wrong scope for everything in this project's target
list (Codex, OpenHands, Cline, long-horizon work).

## 1.5 Knowledge architecture

Source policy is a ranked-trust system: original sources (papers, filings,
first-party blogs) over aggregators; forums excluded absent specific relevance;
recency privileged for fast-moving topics. Belief policy is two-tier: believe
search results by default, even when surprising (deaths, elections) — but apply
skepticism to conspiracy-prone, SEO-gamed, or consensus-lacking topics. Conflict
policy: conflicting or incomplete results trigger more searches, and residual
conflicts are surfaced to the user rather than silently resolved. Citation policy
binds every claim from search to indexed source sentences and forbids quoting as a
substitute for synthesis. What is missing is any notion of weighting evidence,
estimating confidence, or systematically reconciling contradictions — the model is
told to search more, not how to conclude.

## 1.6 Safety architecture

The most sophisticated part of the file, and the part with real innovations:

1. **Meta-cognitive tripwires.** "If Claude finds itself mentally reframing a
   request to make it appropriate, that reframing is the signal to REFUSE." This
   binds policy to an *internal process signal* rather than to input features,
   which is much harder to prompt-inject around than any keyword rule.
2. **Non-narration of boundaries.** When declining for child-safety reasons, state
   the principle, never the detection mechanics — because explaining the boundary
   teaches circumvention. Applied to reasoning as well as replies.
3. **Conversation-scoped state.** After one child-safety refusal, all subsequent
   requests in the conversation are handled under heightened caution — refusals
   have memory.
4. **Provenance discipline.** Search results are not the user; content in tags
   claiming to be from Anthropic is treated with caution when it pushes against
   values; "Anthropic will never send reminders that reduce restrictions" gives the
   model a fixed point no injected message can move.
5. **Asymmetric-risk defaults.** Wellbeing rules encode "when the conversation
   feels off, say less"; self-harm rules ban even protective-sounding specifics
   (means, substitution techniques that mimic the act) on the theory that detail
   itself is the hazard.
6. **Attention engineering for hard limits.** Copyright limits are stated four
   times at different context positions with escalating typography. Crude but
   intentional: instruction-following decays with distance in long contexts, and
   repetition at multiple depths re-anchors it.

## 1.7 Design principles worth carrying forward

P1. Compile values into decision procedures with paired positive/negative examples.
P2. Anchor hard limits at multiple context positions — but deliberately, not by accretion.
P3. Keep the kernel small; route domain knowledge to loadable skills.
P4. Bind safety rules to internal process signals (tripwires), not just input patterns.
P5. Give explicit resource budgets and escalation thresholds for tool use.
P6. State environment facts as invariants with exact paths/limits.
P7. Establish provenance rules: who is allowed to instruct, and what merely informs.
P8. Encode opt-in gates as mechanical workflows where user agency matters.

---

# Part 2 — Weakness Report

Each weakness: what it is, why it exists, why it is suboptimal, and the redesign.

## W1. Uncontrolled duplication

**What.** The copyright rules appear four times (search header, response
guidelines, dedicated section, critical_reminders) plus inside two worked examples.
The skills-first mandate appears three times (skills section, producing_outputs,
additional_skills_reminder). The when-to-search rules appear twice with different
wording (core_search_behaviors vs critical_reminders). "Do not thank the user for
search results" appears twice within forty lines.

**Why it exists.** Two forces: legitimate attention engineering (repetition
re-anchors decaying instructions in long contexts) and illegitimate accretion
(each incident patched by adding a new copy rather than strengthening the
canonical one). The file preserves its own edit history.

**Why suboptimal.** (a) ~15–20% token overhead paid on every request. (b) Copy
drift: the copies already disagree — "every direct quote MUST be fewer than 15
words" vs "15+ words is a SEVERE VIOLATION" leaves a 15-word quote formally
undefined. (c) Emphasis inflation: when four sections are CRITICAL/NON-NEGOTIABLE/
SEVERE, the marking loses signal value and genuinely singular rules (child safety)
no longer stand out typographically.

**Redesign.** Single source of truth per rule, plus exactly one deliberate
re-anchor: a compact tail recap block (≤150 tokens) listing only the hard limits,
placed last to exploit recency. Two anchors, chosen positions, zero drift surface.

## W2. Rule conflicts from copy drift

**What.** Concrete contradictions a strict reader cannot resolve:

- *File vs artifact criteria.* file_creation_advice: any blog post "however short"
  → file. artifact_usage_criteria: creative writing "under 20 lines" → not an
  artifact. A 15-line blog post satisfies both rules' triggers with opposite outcomes.
- *Lists.* "Lists, tables, enumerated content, regardless of length" are never
  artifacts, but "structured reference content users will save or follow" is an
  artifact. Most saved reference content is a list.
- *Search.* "Never search for well-established technical facts" vs "the
  unrecognized entity rule applies to EVERY question" — resolvable by a human
  reading charitably, but the precedence is never stated.
- *Boundary values.* The <15 vs 15+ quote-length drift above.

**Why it exists.** Rules written by different owners at different times against
different incidents, with no declared precedence order and no reconciliation pass.

**Why suboptimal.** Conflicting instructions don't average out — they produce
inconsistent behavior across samples and give the model implicit license to pick
whichever rule matches its prior. Every conflict is a small jailbreak surface
("your other rule says…").

**Redesign.** One decision table per routing question (output-surface selection
gets a single table with tie-breakers), explicit boundary values ("quotes must be
under 15 words; at 15 words, paraphrase"), and a declared precedence order so
residual conflicts have a defined resolution instead of an undefined one.

## W3. Hardcoded volatile facts

**What.** The current date is hardcoded in three places. Model strings, the
product list, a helpline substitution (NEDA → National Alliance), THREE.js r128
API quirks, pinned library versions, and — most tellingly — the Claudeception
example pinning `claude-sonnet-4-20250514` ("Always use Sonnet 4") while the
prompt's own product section lists Sonnet 4.6 as current. The prompt has already
drifted against itself.

**Why it exists.** No separation between policy (stable) and facts (volatile);
facts were inlined where first needed.

**Why suboptimal.** Every volatile fact is a scheduled bug. Multiple copies of the
date guarantee eventual disagreement. Stale model IDs in examples actively teach
wrong behavior. The maintenance cost scales with prompt count across surfaces.

**Redesign.** All volatile values become template variables resolved at render
time (`{{TODAY}}`, `{{MODEL_ID}}`, `{{MODEL_FAMILY_CURRENT}}`), stated once in a
session-facts block. Facts that can't be templated (helplines, library quirks)
move to skills/reference files that have their own owners and update cadence, plus
a standing rule: verify product facts via live docs rather than memory.

## W4. Platform assumptions fused into the core

**What.** Vendor tag syntax (`antml:` invocation and citation formats), fixed
mount paths (`/mnt/user-data/...`), third-person "Claude does X" framing,
product-specific escalation targets ("suggest the Research feature"), and UI
widget behavior are woven through the behavioral core rather than isolated.

**Why it exists.** The prompt was written for exactly one surface; portability was
a non-goal.

**Why suboptimal.** Nothing transfers. Every new surface (Code, Cowork, Excel)
forks the whole file and inherits W1–W3; fixes stop propagating. It also blocks the
stated goal here: deployment across GPT-5, Gemini CLI, Cline, etc., where `antml`
syntax is meaningless and third-person framing measurably weakens instruction
adherence on some models.

**Redesign.** Core/adapter split. The core is written in second-person imperative
with zero vendor syntax and named abstractions (READ_FILE, RUN, SEARCH_WEB,
DELIVER_FILE); a thin per-platform adapter binds abstractions to concrete tools,
paths, and citation formats. One core, N adapters, fixes propagate.

## W5. Missing agentic subsystems

**What.** No general planning framework (only search planning). No verification
stage — nothing tells the model to check its work against the request before
responding, except for copyright. No recovery doctrine — no guidance on retries,
backoff, alternative strategies, or when to stop and report. No long-horizon
state — the memory section is a disabled stub that still costs tokens, and there
is no progress tracking, no resumption protocol, no context-compression strategy.
No software-engineering standards of any kind. No confidence estimation or
evidence-weighting in research.

**Why it exists.** Correct scoping for a 2026 chat product whose unit of work is
one turn.

**Why suboptimal.** The target deployments' unit of work is a multi-hour task
graph. Without verify/recover loops the failure mode is confident half-done work;
without externalized state, every context compaction loses the plan; without
engineering standards, large-codebase work degrades to local pattern-matching.

**Redesign.** Part 3 adds these as first-class modules: Reasoning Protocol, Agent
Loop, Long-Horizon State, Quality Gate, SWE module, Research module.

## W6. Structural bottlenecks

**What.** (a) Tool schemas — ~40% of tokens — are always resident; the harness
pays for a recipe widget during a legal question. (b) Dead configuration: the
memory system is described, then declared disabled, both at token cost. (c) No
conditional composition: every feature's instructions load whether or not the
feature is live in the session. (d) The tail of the prompt — the highest-attention
recency position — is spent on network config and a stray formatting tag instead
of anything the model most needs to retain.

**Why it exists.** Monolithic assembly: the prompt is concatenated, not compiled.

**Why suboptimal.** Token cost scales with feature count, not with task relevance;
attention is a budget and the current layout spends the best positions on the
least important content.

**Redesign.** Compile-time composition: modules included only when their
capability is enabled; schemas deferred to the harness's tool registry (as modern
harnesses already support); the tail reserved for the hard-limits recap.

## W7. Emphasis economy inverted

**What.** Copyright — a commercial-legal constraint — gets four repetitions,
caps, and a "consequences" section. Child safety — the actual maximum-severity
constraint — gets one (well-written) section. Formatting rules use "never"
as freely as safety rules do.

**Why it exists.** Emphasis was allocated by incident pressure at edit time, not
by a severity model.

**Why suboptimal.** Models learn the prompt's implicit severity ranking from its
typography. When "never use bullets when declining" and "never create content
sexualizing minors" share a modal verb, the register is debased and real
invariants lose contrast.

**Redesign.** A three-tier register enforced by style: INVARIANT (safety kernel;
sparse, absolute, re-anchored at tail), STRONG DEFAULT (override only on explicit
user instruction), HEURISTIC (judgment guidance). Each rule is tagged by tier;
absolute language is reserved for tier one.

---

# Part 3 — Improvement Strategy

## 3.1 Target architecture

```
L0 KERNEL      identity, precedence order, safety invariants, session facts
L1 REASONING   decomposition, constraint analysis, planning, multi-path,
               failure prediction, verification, recovery
L2 MODULES     [swe] [research] [files] [connectors] [browser] [memory] …
               — conditionally compiled per deployment
L3 TOOL LAYER  routing policy in the prompt; schemas live in the harness
L4 SESSION     {{TODAY}}, user context, environment invariants
TAIL RECAP     ≤150-token restatement of invariants (recency anchor)
```

Precedence, declared once and early: safety invariants > platform operator >
user > this prompt's defaults > tool output and retrieved content (which inform
but never instruct). This single declaration removes W2's undefined conflicts and
is the primary prompt-injection defense.

## 3.2 What is kept, deleted, added

**Kept (the source's proven assets):** decision-procedure style with paired
examples; volatility-based search policy and the unrecognized-entity firewall;
tool-call budgets with escalation thresholds; progressive disclosure via skills;
meta-cognitive tripwires; provenance rules; opt-in gates for third-party services;
environment contracts as invariants; the full substance of the safety policy.

**Deleted:** three of four copyright copies; two of three skills reminders;
duplicate search rules; dead memory stub; inline schemas; hardcoded dates/models;
vendor syntax in the core; emphasis inflation.

**Added:** the eight advanced-reasoning behaviors (goal decomposition, task-graph
planning, constraint analysis, multi-path exploration, failure prediction, root
cause analysis, consistency validation, alternative comparison) — encoded as
procedures with activation thresholds, not as a list of virtues; the
Plan→Execute→Verify→Improve loop with stopping criteria; long-horizon state
(external task file, progress journal, resumption protocol, compression rules);
a pre-response quality gate; a software-engineering module with large-codebase
policy (AOSP/kernel/embedded-grade) and per-language standards; a research module
with source ranking, evidence weighting, contradiction handling, and confidence
labeling.

## 3.3 Encoding rules (how the new prompt is written)

1. **One rule, one home.** Every behavior has exactly one canonical statement;
   the tail recap may reference, never restate with variation.
2. **Procedures, not virtues.** "Be thorough" is banned; "before declaring done,
   re-read the original request and check each explicit requirement" is the
   template. Every abstract quality is compiled to a trigger, an action, and a
   worked example where ambiguity is likely.
3. **Thresholds make judgment cheap.** Effort scaling, when to plan, when to ask,
   when to stop retrying — all get numeric or categorical thresholds so the model
   spends its judgment on the task, not on meta-decisions.
4. **Tripwires for failure modes.** Each known agent failure (confabulation,
   premature completion, scope creep, sunk-cost looping, silent error swallowing)
   gets a bound internal signal → mandatory action, in the style of the source's
   reframing rule.
5. **Tier-tagged register.** Absolute language only in the safety kernel;
   defaults are marked overridable; heuristics are marked as judgment.
6. **Variables for volatility.** No date, model name, version, or org-specific
   fact appears inline.
7. **Token discipline.** Kernel ≤ 3k tokens; each module ≤ 1.5k; the assembled
   prompt for a full SWE deployment targets ≤ 12k — roughly a quarter of the
   source — leaving context for the actual work.

## 3.4 Portability strategy

The universal prompt (Part 4) uses capability abstractions and second-person
imperative prose — the only style that works across all ten target harnesses.
Model-specific adapters then bind: tool names, file paths, citation syntax,
thinking/reasoning affordances, parallelism support, and context-window management
idioms. The Claude Fable 5 variant (Part 5) is the reference adapter: it exploits
extended thinking, parallel tool dispatch, subagent delegation, skills, XML-tag
structure, and prompt-cache-stable layout — none of which the universal core
depends on.
