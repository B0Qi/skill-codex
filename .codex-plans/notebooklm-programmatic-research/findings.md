# Findings and Decisions

## Requirements
- Deliver report at `/tmp/notebooklm-research-codex.md`.
- Research-only task; do not modify existing repository code.
- Required topics:
- `notebooklm-py` maintenance/stability and community signals.
- Other programmatic clients (`nlm`, `nblm-rs`, plus any new candidates).
- MCP server options and maintenance/stability.
- Official Google API roadmap for NotebookLM/Q&A endpoints.
- Alternative approaches (Gemini API grounding, AI Studio, related patterns).
- Final recommendation prioritizing stability.

## Research Findings
### Initial source map (2026-02-12)
- `teng-lin/notebooklm-py` identified on GitHub and PyPI; PyPI shows rapid release cadence in Jan 2026 (0.1.1 -> 0.3.2).
- `tmc/nlm` (Go CLI) identified as NotebookLM command-line tool.
- `K-dash/nblm-rs` identified as Rust + Python client targeting NotebookLM Enterprise API, explicitly claiming Sept 2025 API release.
- `PleasePrompto/notebooklm-mcp` identified as browser-automation MCP integration.
- Google Cloud/Agentspace release notes indicate official NotebookLM Enterprise API milestones:
- 2025-09-05: notebook creation/management API GA.
- 2025-08-28: podcast API GA with allowlist.

### Immediate implications
- Ecosystem appears split between:
- unofficial consumer-web reverse-engineering (likely brittle), and
- official enterprise API wrappers (newer but potentially more stable).
- Need deeper validation of maintenance signals (issue velocity, unresolved breakages, release recency).

## Technical Decisions
| Decision | Rationale |
|----------|-----------|
| Use web + primary artifacts (GitHub repos/issues/releases, docs) as evidence backbone | Minimizes rumor/error risk for fast-moving ecosystem topics. |
| Treat community posts (Reddit/HN) as anecdotal risk signal, not source of truth | Better stability assessment quality. |
| Separate "consumer NotebookLM automation" from "NotebookLM Enterprise API" | Stability/risk profile differs materially. |

## Issues Encountered
| Issue | Resolution |
|-------|------------|
| Some search results are noisy/non-authoritative | Will prioritize direct repo/docs pages and official Google sources. |

## Resources
- https://github.com/teng-lin/notebooklm-py
- https://pypi.org/project/notebooklm-py/
- https://github.com/tmc/nlm
- https://github.com/K-dash/nblm-rs
- https://github.com/PleasePrompto/notebooklm-mcp
- https://cloud.google.com/agentspace/docs/release-notes

### notebooklm-py: repo-level signals
- GitHub snapshot (captured 2026-02-12): ~1.9k stars, ~190 forks, 460 commits, 3 open issues.
- Open issues include automated `RPC ID Mismatch Detected` with `rpc-breakage` label (opened 2026-02-09).
- Closed issues list shows prior `RPC ID Mismatch Detected` incidents closed on 2026-01-22 and 2026-01-19, indicating recurring break/fix episodes.
- Closed issues are heavily concentrated in Jan 2026 (cross-platform auth/runtime fixes and feature correctness), suggesting active maintenance plus rapid change surface.
- PyPI release cadence for `notebooklm-py` is very high in Jan 2026 (0.1.1 -> 0.3.2 within ~17 days).

### Interpretation note
- Preliminary inference: maintenance appears active and responsive, but underlying integration is brittle due dependence on undocumented endpoints and recurring RPC mismatch breakages.

### notebooklm-py: detailed stability signals
- GitHub API confirms 22 issues and 90 PRs total; only ~4 listed contributors, indicating high bus-factor concentration.
- `rpc-breakage` label appears on 3 issues (2026-01-19, 2026-01-22, 2026-02-09), with two historically fixed within hours the same day.
- Commit history is extremely bursty: 460 commits between 2026-01-05 and 2026-01-27, then no pushes after 2026-01-27 as of 2026-02-12.
- Release stream is rapid (`v0.1.4` to `v0.3.2` across Jan 2026).
- `v0.3.0` includes migration guidance, deprecations slated for removal in `v0.4.0`, and removed APIs (`detect_source_type`, `ARTIFACT_TYPE_DISPLAY`), confirming meaningful breaking surface.
- `v0.2.1` explicitly adds nightly RPC health checks that auto-file `rpc-breakage` issues, which partially explains visible breakage frequency.

### notebooklm-py: preliminary risk profile
- Strength: fast maintainer response and high implementation velocity.
- Weakness: unofficial endpoint coupling causes recurring compatibility incidents; APIs still evolving quickly with deprecation/removal churn.

### Other programmatic clients
- `tmc/nlm` (Go CLI):
- 274 stars / 36 forks; 99 commits total from 2024-11-18 to 2025-09-19.
- 11 issues total (9 open, 2 closed); latest closed issue dates to 2025-12-18 while latest open issue is 2026-02-01.
- No GitHub releases; recent development appears paused since Sept 2025 despite continued issue intake.
- README indicates browser-auth/token extraction approach for consumer NotebookLM.
- `K-dash/nblm-rs` (Rust CLI + Python `nblm`):
- 75 stars / 7 forks; 101 commits from 2025-10-19 to 2026-02-08.
- README explicitly targets **NotebookLM Enterprise API** only and states consumer API is unpublished.
- 9 releases, latest `v0.2.3` (2025-11-19); PyPI `nblm` latest also `0.2.3` (2025-11-19).
- Issues are low and mostly enhancement/docs; no visible stream of breakage incidents tied to RPC mismatch.
- Recent pushes are mostly dependency/CI maintenance (including Renovate bot), so feature velocity is moderate.
- Additional/new client spotted: `photon-hq/notebooklm-kit` (TypeScript SDK, 23 stars).
- Uses cookie-based auth flow; README references internal operations and credential persistence.
- Last push 2026-01-14; small community footprint and likely same unofficial-API fragility class as `notebooklm-py`.

### MCP server landscape
- `jacob-bd/notebooklm-mcp-cli` currently appears most active among high-adoption MCP options:
- 949 stars / 247 forks; pushed 2026-02-12; recent release `0.2.20` with active issue churn.
- README clearly discloses use of undocumented internal APIs and warns they may change without notice.
- `PleasePrompto/notebooklm-mcp` has high stars (872) but no pushes since 2025-12-27 and many open issues (13/19).
- `khengyun/notebooklm-mcp` appears mostly stale (last push Oct 2025) with only open issues.
- `Pantheon-Security/notebooklm-mcp-secure` shows recent maintenance (push Feb 2026) but is a forked variant with small community (26 stars), so longevity risk is uncertain.

### Community signals (Reddit/HN)
- Hacker News has recent Show HN posts for both `notebooklm-py` (2026-01-12) and `nblm-rs` (2025-10-27), but engagement is low (single-digit/low-teens points), indicating early-stage ecosystem maturity.
- Reddit search and related thread evidence shows repeated demand for official NotebookLM APIs and community workarounds; anecdotal but consistent with unofficial-client volatility.

### Emerging interpretation
- Unofficial consumer NotebookLM integrations are numerous and fast-moving, but maintenance durability varies widely.
- Enterprise-API-based tooling (`nblm-rs`) appears structurally more stable even when less feature-dense for consumer workflows.

### Official Google API roadmap signals (NotebookLM Enterprise API)
- Google Cloud documentation now provides explicit REST resources/methods for NotebookLM Enterprise API, including:
- `projects.locations.notebooks.chat` (notebook-level Q&A/chat)
- `projects.locations.notebooks.sources.chat` (source-level Q&A/chat)
- Release notes (official) indicate staged rollout:
- 2025-08-28: Enterprise API entered Preview.
- 2025-09-05: Notebook creation and management reached GA.
- 2025-09-23: Notebook and source chat endpoints entered Preview.
- 2026-01-27: Notebook and source chat endpoints reached GA.
- No public source found with forward-dated roadmap commitments beyond release-note entries; public signal is feature-by-feature release notes rather than long-horizon roadmap docs.

### Alternative approaches (official/less brittle)
- Gemini API now exposes multiple grounding/research primitives that can replace some NotebookLM usage:
- Google Search grounding tool
- URL context tool (single URL per request; includes URL-context metadata)
- File Search + vector stores
- (Preview) Deep Research workflow support
- Google AI Studio provides a fast prototyping path for these capabilities without standing up custom infra first; useful for validating memory-notebook UX before production hardening.

### Practical synthesis
- For production stability, official Enterprise API and official Gemini tools offer materially lower breakage risk than consumer NotebookLM reverse-engineering.
- Unofficial clients/MCP remain useful for consumer-account workflows where Enterprise API is unavailable, but should be treated as volatile adapters.

## Final recommendation snapshot
- Most stable path: official NotebookLM Enterprise API where available.
- Next-best stable alternative: Gemini API grounding stack (Search/URL/File Search) + AI Studio prototyping.
- Consumer NotebookLM unofficial clients should be wrapped as best-effort adapters with operational safeguards.
