# Global Resource Flow Intelligence Platform

Global resource flow visualization platform. Aggregates worldwide resource statistics, infers time-series flows, and renders interactive visualizations.

## Live

- https://global.etzhayyim.com

## MCP Endpoint

```
POST https://actors.etzhayyim.com/{nanoid}/api/mcp
Content-Type: application/json

{"jsonrpc":"2.0","method":"tools/list","id":1}
```

### Tools

| Tool | Description |
|---|---|
| `global.list_resources` | List global resources |
| `global.list_flows` | List resource flows (filter by resource_id, year) |
| `global.get_graph` | Build resource graph for 3D visualization |
| `global.get_resource_stats` | Get region stats for a resource |
| `global.get_timeline` | Get timeline data for a resource |
| `global.list_systems` | List system models |
| `global.get_system` | Get a full systems-thinking model |

## Project Structure

```
PROJECT.jsonld              # Project metadata (JSON-LD)
scheduler.jsonld            # Milestone tracking
MCP_TOOLS.md                # MCP tool contract documentation
README.edn                  # Machine-readable self-description
migration.edn               # Provenance: what was extracted, and from where
facts/catalog.edn           # Cited public sources, with a declared floor
tools/verify_citations.cljk # Live gate: every citation must return 2xx
tools/check_citation_status.cljk # Weekly manual status check (AWAI-3824)
receipts/                   # Dated status receipts, one file per weekly run
test/run.cljk               # Offline gate: the records parse and agree
```

Every path above exists — `test/public_global/records_test.cljk` reads this
block and fails if one does not. It was added because this list previously
described a `wasm/global-mcp-component/` tree that the extraction never
carried (`migration.edn` records `:go-files-created 0` in the same directory),
so a reader followed the README to files that had never been here.

**The wasmCloud MCP component is not in this repository.** The tool contract it
implements is `MCP_TOOLS.md`; the component itself stayed behind in
`etzhayyim/root` at the revision `migration.edn` names.

## Gates

```bash
kbb --backend sci --classpath "test:." test/run.cljk        # offline: parses, shapes, this README
kbb --backend sci tools/verify_citations.cljk               # live: every cited URL returns 2xx
```

The two are separate because they answer different questions and fail for
different reasons. The live gate is worth what the network is worth at the
moment it runs; the offline one settles what the bytes here can settle.

## Citation status (weekly manual gate, AWAI-3824)

Decision (AWAI-3824, owner 2026-11-22): "(b) 手動の検査。CI の例外の
runner は作らない。" Tatara CI runs with `:network :none`, so the live 2xx
gate (`tools/verify_citations.cljk`) cannot run there. Leaving it red in CI
is not an option, and silently skipping it is the failure this policy exists
to prevent.

- Who runs it: once a week, an Orca automation (Ontology Collector) wakes a
  Muse worker to run the check and commit the dated receipt in a small
  one-commit PR.
- Command: `kbb --backend sci tools/check_citation_status.cljk` (writes
  `receipts/<UTC-date>.edn`; re-check an old claim with `kbb --backend sci
  tools/check_citation_status.cljk --verify receipts/<date>.edn`, which fails
  when the receipt is missing).
- Method: HEAD only per URL; one Range GET retry (`bytes=0-0`) when HEAD
  answers 4xx/5xx or fails at the transport level; at least 1000 ms between
  two requests to the same host; generic User-Agent
  (`Mozilla/5.0 (compatible; citation-status-check/1.0)`, no personal name,
  no mail address).
- Recorded: per-URL status class (2xx / 3xx / 4xx / 5xx / error) plus summary
  counts and the date. Response bodies, substrings and archives are never
  stored. Citations are URLs with live status only; no third-party
  classification text or definitions are republished here.
- Last checked: 2026-11-22 (`receipts/2026-11-22.edn`: 27 x 2xx
  direct, 1 x 4xx primary verified via its check URL, 1 bot-blocked).
- Bot-blocked (AWAI-3824 follow-up, owner decision (b)): a cited URL that
  answers 403 to both HEAD and Range GET because of a bot wall — not a move
  and not dead — keeps its canonical `:cite/url` and carries the mark
  `:cite/bot-blocked true` plus a machine-checkable `:cite/check-url`
  (an `api.worldbank.org/v2/...?format=json` URL from the same source
  family). The checker probes both with the same method; a 2xx on the
  check URL counts as verified (`:via-check-url true`, primary status kept
  in the receipt). Red only when both fail, naming both causes. The
  receipt keeps the bot-blocked count separately (`:receipt/bot-blocked`).
  Current: `:worldbank/data-portal` (`https://data.worldbank.org/`,
  Cloudflare challenge, human-verified alive 2026-11-22) is checked via
  `https://api.worldbank.org/v2/country?format=json&per_page=1`.
- No circumvention, ever: the generic User-Agent above is the only
  identity sent; challenge pages are never solved and no headless browser
  is used. A bot wall is recorded (`:cite/bot-blocked`), never bypassed.
- Triage: a non-2xx turns the gate red with the URL and cause named.
  Re-check the URL, mark it moved, or replace the citation; never delete a
  failing citation to make the gate green. URLs whose 4xx/5xx is new since
  the previous week are listed in a comment on AWAI-3824.

New files in this change (`tools/check_citation_status.cljk`, `receipts/`,
this section) are our own work; their license awaits the rights holder's
decision. No new LICENSE was added.

## License

Apache License 2.0 — see [LICENSE](LICENSE).
