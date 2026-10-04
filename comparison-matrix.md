# Comparison matrix

Running ten-dimension comparison across every harness in the series.
Each column is grounded in that article's pinned source tree and endnotes.

| # | Dimension | pi (v1.0.2) | aider (v0.86.2) |
|---|---|---|---|
| 1 | Agent loop | Event-sourced; inner (tool/steer) + outer (follow-up) loops; no iteration cap [^5][^6][^10] | No tool-call loop; parse→apply→reflect over text edit formats; REPL + reflection (≤3) + retry; no REPL iteration cap [^4][^6][^8] |
| 2 | Tool system | 4 built-ins + registry; TypeBox validation; sequential/parallel; nested calls [^22][^27][^28] | None — model emits SEARCH/REPLACE blocks parsed by edit-format coders; shell suggestions run only with per-command confirmation [^14][^18] |
| 3 | Model providers | ~35 behind one `StreamFn`; per-request key resolution [^18][^29] | litellm behind a `Model` wrapper; 313-entry YAML + substring heuristics (model-aware, load-bearing); weak model for commits/summaries [^25][^26] |
| 4 | Prompt construction | Structured sections, diffed per turn, XML-tagged [^15][^16] | Fixed wire order with synthetic user/assistant pairs; prompt-cache breakpoints on stable prefixes; repo map re-injected per turn [^10][^11] |
| 5 | Memory/session | JSONL sessions; compaction with branch summarization [^30] | In-memory cur/done messages; background-thread summarization; markdown log in repo; git auto-commit per turn [^16][^27] |
| 6 | Reasoning/planning | Thinking levels forwarded; no planner/sub-agents (omitted) [^29][^32] | Architect = sequential 2-model delegation with user gate; no autonomous sub-agents [^24] |
| 7 | Extensibility | TS extensions, hooks at every seam, MCP, skills [^31] | 42 closed slash commands; model-settings YAML; no plugin API, MCP, or skills — configuration, not code [^32][^33][^34] |
| 8 | Interfaces | TUI / print / RPC / SDK — all event subscribers [^9] | prompt_toolkit CLI; `--message` one-shot; streamlit GUI; imperative rendering, no event stream [^20] |
| 9 | Failure handling | Auto-retry, truncation guards, abort; no iteration cap [^21][^26][^10] | Exp-backoff retry (60s cap); no retry on context exhaustion; malformed edits reflected (≤3); lint/test confirm loops [^7][^18] |
| 10 | Security model | Project trust + extension hooks; no built-in approval UX [^24][^33] | Chat-file edits auto-apply; prompts only at boundaries; git auto-commit + `/undo`; no sandbox [^30] |

## Columns

- **pi (v1.0.2)** — [article](pi/article.md) · source: [earendil-works/pi](https://github.com/earendil-works/pi)
- **aider (v0.86.2)** — [article](aider/article.md) · source: [Aider-AI/aider](https://github.com/Aider-AI/aider)
