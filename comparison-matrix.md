# Comparison matrix

Running ten-dimension comparison across every harness in the series.
Each column is grounded in that article's pinned source tree and endnotes.

| # | Dimension | pi (v1.0.2) |
|---|---|---|
| 1 | Agent loop | Event-sourced; inner (tool/steer) + outer (follow-up) loops; no iteration cap [^5][^6][^10] |
| 2 | Tool system | 4 built-ins + registry; TypeBox validation; sequential/parallel; nested calls [^22][^27][^28] |
| 3 | Model providers | ~35 behind one `StreamFn`; per-request key resolution [^18][^29] |
| 4 | Prompt construction | Structured sections, diffed per turn, XML-tagged [^15][^16] |
| 5 | Memory/session | JSONL sessions; compaction with branch summarization [^30] |
| 6 | Reasoning/planning | Thinking levels forwarded; no planner/sub-agents (omitted) [^29][^32] |
| 7 | Extensibility | TS extensions, hooks at every seam, MCP, skills [^31] |
| 8 | Interfaces | TUI / print / RPC / SDK — all event subscribers [^9] |
| 9 | Failure handling | Auto-retry, truncation guards, abort; no iteration cap [^21][^26][^10] |
| 10 | Security model | Project trust + extension hooks; no built-in approval UX [^24][^33] |

## Columns

- **pi (v1.0.2)** — [article](pi/article.md) · source: [earendil-works/pi](https://github.com/earendil-works/pi)
