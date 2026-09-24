# @pipeworx/bold-systems

The global DNA barcode reference library from BOLD Systems (Barcode of Life Data System, Centre for
Biodiversity Genomics, University of Guelph) — specimen vouchers with locality, institution and
taxonomy, the COI/rbcL/matK barcode sequences themselves, and BIN species-proxy clusters.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1679+ live data sources.

## Tools

- `bold_specimens(taxon?, country?, bin?, institution?, query?, limit?, offset?)` — specimen records:
  process ID, full taxonomy, BIN, collection date, locality with coordinates, holding institution,
  marker and GenBank accession. Filters AND together.
- `bold_sequences(taxon?, country?, bin?, institution?, marker?, query?, limit?, offset?)` — the
  barcode sequences, with primers, base count, INSDC accession and a ready-to-paste FASTA block.
- `bold_taxonomy(name, rank?)` — resolve a name against BOLD's own index: lineage, record/species/BIN
  counts at each rank above it, a plain-language description, and alternative matches with counts.

## Auth

Keyless.

## Data sources

- <https://portal.boldsystems.org/api/terms> — term index; resolves a plain name to a query triplet.
- <https://portal.boldsystems.org/api/query> — turns a triplet query into a `query_id`.
- <https://portal.boldsystems.org/api/documents/{query_id}> — the records, `nuc` sequence inline.
- <https://portal.boldsystems.org/api/taxonomy/{description,hierarchy}> — lineage, counts, prose.
- Swagger: <https://portal.boldsystems.org/api/docs> (the spec itself is at
  <https://portal.boldsystems.org/openapi.json>, NOT under `/api/`).

## Traps

**The v4 API is dead.** `v4.boldsystems.org/index.php/API_Public/{specimen,sequence,combined}` — what
every tutorial and most R packages still point at — answers with a "BOLD Public Offline" HTML page,
not JSON, so a naive client reports a parse error rather than a retirement. Everything here is the
v5 portal API.

**The query language is `scope:field:value` triplets and the scope/field pair is not guessable from
the value.** `Canada` is `geo:country/ocean`, `Danaus` is `tax:genus`, `BOLD:AAA9566` is `bin:uri`,
`Smithsonian Institution` is `inst:name`. Free text gets a 400 `{"detail":"Invalid triplet token"}`.
The pack resolves every typed argument through `/api/terms` first, which is what turns a plain name
into a query that matches instead of a 400 or a silent zero.

**`/api/counts` reports the SUM of the per-term counts, not the size of the intersection.** For
`tax:species:Danaus plexippus;geo:country/ocean:Canada` it says **2,520,553** records; the actual
match set is **7**. That is a clean 200 carrying a number wrong by five orders of magnitude, and it
is the number a naive integration would quote. The pack does not use it — `matchingRecords` comes
from `recordsTotal` on the document query, which is the real count. (Semicolon-separated triplets
AND on `/documents` even though they sum on `/counts`.)

**`/api/query/preprocessor` falls back to ID-field matching on a multi-word string.** Feeding it
"Danaus plexippus Canada" returns `ids:processid:...;ids:sampleid:...;ids:insdcacs:...` — a query
that matches nothing and errors nowhere. Resolve each filter separately, which is what
`resolveTriplet` does.

**BOLD holds specimen records that were never sequenced.** `bold_sequences` filters to records with a
`nuc` field and says so in `note` when the result is empty, so "this taxon has no barcodes" is
distinguishable from "this taxon has no records".

**The marker filter is applied to the returned page, not upstream** (BOLD's term index has no entry
for `COI-5P`). The pack over-fetches 5× when a marker is set and reports which markers were actually
present in the page it scanned, so an empty result names the markers you could have asked for.

**`identifier_email` is dropped before anything leaves the pack** — it is a contact address on a
person, not specimen data. Collector and identifier NAMES are kept: those are the published
attribution on a museum voucher and appear on every biodiversity record.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "bold-systems": {
      "url": "https://gateway.pipeworx.io/bold-systems/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/bold-systems/mcp` returns the tools in the table
above **plus the shared Pipeworx meta-tools** — `ask_pipeworx`,
`discover_tools`, `search_within`, `remember`/`recall` and the rest of the
gateway-wide set. So the tool count you see is larger than this table: a
single-pack endpoint currently lists roughly 30 shared tools alongside the
pack's own. The connection's `initialize` response states its exact scope, and
is the authoritative answer for a given day.

This is deliberate, not multiplexing by accident. The meta-tools are what let a
scoped connection answer a question this pack does not cover — via
`ask_pipeworx`, which routes across the whole catalog — without you adding a
second MCP server. There is currently no way to mount a pack endpoint without
them; if the extra schemas cost you more context than the routing is worth,
connect to the full gateway once rather than to several pack endpoints.

Or connect to the full Pipeworx gateway to get every pack's tools listed
directly, instead of just this one's:

```json
{
  "mcpServers": {
    "pipeworx": {
      "url": "https://gateway.pipeworx.io/mcp"
    }
  }
}
```

Both URLs reach the same gateway and the same 1679+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/bold_specimens \
  -H 'Content-Type: application/json' \
  -d '{"taxon":"Danaus plexippus","country":"Canada","limit":3}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/bold_specimens`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "bold-systems": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-bold-systems"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-bold-systems
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Bold Systems data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
