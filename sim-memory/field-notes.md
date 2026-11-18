# Field Notes: The harness is not the memory — Self-Improvement Memory and the optimizer's notebook

**Series framing.** This is a Field Notes piece, not a numbered teardown.
Every teardown in this series dissects a system that runs an agent. The
subject here runs nothing the reader can install: it is a paper — Yu et al.,
*Self-Improvement Memory Enables Continual Recursive Self-Improvement*
(Google Cloud AI, Google DeepMind, Penn State) — and it releases no code.
What it offers instead is the first controlled evidence on a question two
earlier pieces in this series approached from opposite sides. The Darwin
Gödel Machine teardown (#13) examined a harness that rewrites itself and
keeps an archive of its variants, but no structured account of *why* its
edits worked. The Hy Memory Field Notes examined memory as a write-path
engineering problem *inside* a harness. This paper moves the memory one
level up: not the agent's memory of its tasks, but the **optimizer's memory
of its own edits** — and tests whether that memory, rather than the evolved
harness itself, is the asset that transfers when the model or the benchmark
changes. That is a Field Notes question, and this piece answers it from the
paper's own evidence, graded as paper evidence throughout: nothing here
was verified against source code, because there is none to verify against.

> **Pin:** arXiv:2611.18856v1 [cs.LG], 13 Nov 2026, CC BY 4.0. Read in
> full, including appendices and limitations. Companion source: Yu, Liu,
> Long, Zhang & Xue, *From Harness Optimization to Recursive
> Self-Improvement: A Survey of Language Model Agents*, author working
> revision of 14 Nov 2026 (not peer-reviewed) — the same group's survey,
> published a day later, whose evaluation criteria this piece uses as a
> second lens. The paper text identifies no public code or checkpoint
> release. All experimental numbers below are the authors' own.

## The problem: an evolved harness does not explain itself

Harness optimization — an outer loop in which a proposer suggests edits to
an agent's harness, an evaluator scores them, and a selector keeps the
winners — is usually studied with the policy model and the benchmark held
fixed. The paper names the setting where that stops being true **Continual
Recursive Self-Improvement (CRSI)**: optimization *continues* after the
benchmark changes (cross-dataset CRSI, policy fixed) or after the policy
model changes (cross-model CRSI, benchmark fixed), starting from the
previously evolved harness instead of from scratch.[^1]

The motivating observation is that evolution's gains do not reliably
survive the move, and the evolved artifact cannot say why. Source
optimization raised held-out source scores by 2.75–6.41 points in all six
cross-dataset settings — yet the source-evolved FewShotAll harness starts
target optimization at 72.92, *below* the un-evolved seed's 75.72.[^2] In
the cross-model experiments, an Opus-evolved harness first run on Gemini
scores below Gemini's own seed harness in every domain, by 0.60 to 5.48
points; in the workspace domain it starts at 68.41 against the seed's
73.89.[^3] Continuing to optimize from that harness does not reliably fix
it: harness-only continuation finishes *below a cold start* in three of
the four cross-model domains.[^3] The paper's diagnosis is that the
harness alone leaves three questions unanswered — why was an edit
proposed, what was it expected to fix, and what happened when it was
tested? Source optimization produces exactly this evidence. It just does
not normally travel with the harness.[^1]

## The mechanism: a notebook keyed by problem, not by time

Self-Improvement Memory (SIM) is that evidence, structured. An entry is
keyed by a **root cause** — a recurring harness problem, such as premature
completion — and holds a sequence of attempts against it. Each attempt
records the modification and its rationale, the **predicted** outcome
(which tasks the edit is expected to fix, written down *before*
evaluation) and the **observed** outcome, and provenance: domain,
benchmark, policy model, and round.[^4] Two properties of the record
matter more than its schema:

- **Failures are kept.** Each round, only the selected candidate is
  recorded — including when the edit regresses or fails to fix the tasks
  it predicted it would. Entries whose root-cause text matches are merged
  without dropping their attempt histories, and an entry survives the
  replacement of the very edit it describes.[^4][^5]
- **There is no retrieval.** The proposer receives the complete SIM in
  its system prompt every round, and the selector reads the same record
  when screening candidates. Nothing is ranked, truncated, or forgotten
  by a retriever; both LLM "harness engineers" judge each inherited edit
  against the current failures and may retain, revise, replace, or remove
  the modification while keeping its record.[^4][^6]

The experiment then varies only what crosses the boundary. Four
conditions start target optimization: **Cold Start** (seed harness, no
memory), **Harness Transfer** (evolved harness, no memory),
**Problem-Only Memory** (evolved harness plus the root causes alone), and
**Full SIM** (evolved harness plus the complete record). Starting harness,
target tasks, policy, round counts (10), and candidate budgets (5 per
round) are matched across conditions.[^6]

## The evidence

**Cross-dataset** (six transitions across coding, text classification,
workspace, and engineering seeds): Full SIM finishes highest in all six,
beating Harness Transfer by 2.43–4.35 points and Problem-Only Memory by
0.20–6.06. The ablation is the interesting part: Problem-Only Memory is
*unreliable*, ranging from 1.82 points below Harness Transfer to 2.40
above it. The diagnosis alone is not the asset; the asset is the diagnosis
with its attempts and outcomes attached.[^2]

**Cross-model** (Opus 5.5 → Gemini 3.8 Flash, benchmark fixed): Full SIM
finishes highest in all four domains, beating harness-only continuation by
3.60–11.33 points. The workspace case is the headline: both conditions
start at 68.41; Harness Transfer reaches 73.42, Full SIM reaches
84.75.[^3] Across model *generations* (a Gemini 2.5 Flash → 3.5 Flash →
3.8 Flash → 4 Argon chain on Terminal-Bench 2.1, carrying harness and SIM
forward), continuation beats fresh per-generation optimization in two of
three subsequent generations — and loses on the fourth, 94.8 to
95.6.[^7]

**The load-bearing result is the paired study.** Ten independent
Terminus-2 source runs, each continued on Terminal-Bench 4 with and
without its SIM under matched budgets: SIM finishes higher in 9 of 10
pairs, mean advantage **+2.36 points** (95% CI [1.05, 3.68]; paired
t(9) = 4.06, p = 0.003). One pair favors no-SIM continuation, by
0.61.[^8] Everything else in the paper is, by the authors' own reporting
convention, one complete run per condition — descriptive, not a variance
estimate. The paired study is the piece of evidence that survives that
caveat.

**The mechanism shows up in the cases.** Both coding conditions inherit
the same completion gate from source optimization. With SIM, the proposer
reads the gate's rationale and source outcomes, retains completion
verification, and *extends* it with target-specific checks — feasibility
versus optimality. Without SIM, the run tightens the inherited gate,
twice repairs completion deadlocks of its own making, and misattributes
the gate to its own first round. The requirement transferred; only the
memory condition determined whether its implementation was revised or
thrashed.[^5] Retention tells the same story from the other side: Full
SIM loses 3 of 225 previously successful source rollouts during target
optimization, against 6 for harness-only continuation.[^5]

## The second lens: the survey wrote the contract first

The companion survey — same group, revised one day after this paper
appeared — never names SIM, but its appendix on persistent improvement
evidence prescribes, almost clause for clause, the record SIM implements:
a memory entry should connect "the initiating failure, attempted edit,
positive and negative evaluation evidence, and the conditions under which
reuse is warranted."[^9] Its comparison table likewise demands the
controls this paper mostly runs: retained versus reset improvement state
at matched budgets, with initialization and subsequent improvement
reported separately.[^10] And its conclusion draws the distinction the
whole result hangs on: improving an agent is not the same as improving
how later agents are improved; a retained artifact earns its claim only
when it demonstrably changes later decisions.[^11] Read together, the two
papers are a prescription and its first controlled test — published
together, whether or not they were written that way.

## Limits, on the record

- **Paper evidence only.** No code or checkpoints were released; the
  proposer/selector prompts exist as appendix summaries, and no claim in
  this piece could be checked against an implementation. That is a weaker
  footing than any numbered teardown in this series, and the reason this
  is a Field Note.
- **One run per condition outside coding** — the authors state this
  themselves: independent repetitions were affordable only in the coding
  domain.[^12] The cross-model headline (84.75) is a single run.
- **The engineers are models too.** Proposer and selector are LLMs
  (Gemini 3.8 Flash in the main comparisons); an ablation with Claude
  Opus 5.5 in those roles produced *larger* SIM gains from both coding
  seeds — so the size of the effect depends on who is doing the
  remembering and revising.[^6]
- **Whole-memory prompting does not scale by construction.** Reading the
  entire record every round is a deliberate design choice at this
  history length; the paper does not test where it breaks.
- **No cost accounting across conditions.** The procedure is evaluated
  whole — inherited memory plus continued updating — and the paper
  reports no token or runtime comparison between memory conditions, the
  very joint reporting its companion survey recommends.[^10][^12]

## What this changes

Every harness in this series keeps some account of itself: transcripts,
event logs, skill libraries, archives of variants. The consistent finding
across the teardowns — stated as F2 in the survey paper this site
accompanies — is that what a system *stores* is usually a transcript, and
what it can *use* is knowledge distilled from one. SIM is that distinction
applied to the one actor the teardowns never instrument: the engineer
doing the editing. Its result, at the strength the paired study supports,
is that when a harness must outlive its original model and benchmark, the
transferable asset is not the evolved artifact. It is the notebook —
problems, attempts, predictions, outcomes — that explains how the
artifact came to be.

[^1]: Yu et al., arXiv:2611.18856v1, §1 and §3.1: CRSI definition; the
    three unanswered questions; contributions list.
[^2]: Yu et al., §4.2 and Appendix E.1/E.2: cross-dataset margins;
    FewShotAll 72.92 vs seed 75.72; Full SIM 77.92 vs Harness Transfer
    74.12 vs Cold Start 75.92.
[^3]: Yu et al., §4.3.1 and Figure 4: cross-model starting deficits
    0.60–5.48; workspace 68.41 → 84.75 (Full SIM) vs 73.42 (Harness
    Transfer); margins 3.60–11.33.
[^4]: Yu et al., §3.2 and Figure 2: SIM entry structure (Eq. 2);
    predictions recorded before evaluation; only the selected candidate
    recorded per round, regressions included; entry merging.
[^5]: Yu et al., §4.4 and Appendices F.3–F.6: the completion-gate case;
    diagnosis P3 accumulating target evidence; entry P8 surviving edit
    replacement; retention 3/225 vs 6/225 lost rollouts.
[^6]: Yu et al., §3.2–§3.3 and §4.1: full SIM in the proposer's system
    prompt, no retrieval or truncation; four transfer conditions; N = 10
    rounds, k = 5 candidates; proposer/selector models and the Opus 5.5
    engineer ablation (Appendix E.4.2, Table 7).
[^7]: Yu et al., §4.3.2, Table 1: generation chain on TB2.1; CRSI vs
    Fresh RSI +3.5 (3.5 Flash), +1.6 (3.8 Flash), −0.8 (4 Argon).
[^8]: Yu et al., §4.4, Table 3 and Figure 5: ten paired Terminus-2
    continuations; 9/10; mean +2.36, 95% CI [1.05, 3.68], p = 0.003;
    signed-rank p = 0.006; the −0.61 pair.
[^9]: Yu, Liu, Long, Zhang & Xue, *From Harness Optimization to
    Recursive Self-Improvement* (working revision, 14 Nov 2026),
    Appendix K.7, "A Contract for Persistent Improvement Evidence."
[^10]: Companion survey, Table 1 ("Proposed comparisons for interpreting
    harness-improvement claims") and §5: retained vs reset state at
    matched budgets; retention and total cost reported together.
[^11]: Companion survey, §6 (Conclusion): improving an agent vs improving
    how later agents are improved; evidence must connect the retained
    artifact to later decisions.
[^12]: Yu et al., Limitations: independent repetitions focused on coding;
    other domains one run per condition; SIM evaluated as a complete
    procedure; compatible harness interfaces assumed.
