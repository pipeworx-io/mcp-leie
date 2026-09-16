# leie

Is this healthcare provider barred from federal health programs? Screening
against the HHS OIG List of Excluded Individuals/Entities — the check you make
before paying, employing or contracting with a provider.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1576+ live data sources.

## Tools

| Tool | Answers |
|---|---|
| `check_provider_exclusion` | "Is this provider on the OIG exclusion list?" — candidates with the fields that tell same-named people apart. |
| `provider_exclusion_coverage` | How many exclusions are held, how current the copy is, and why NPI often cannot disambiguate. |

## Auth

None. No key, no account.

## Data source

**HHS Office of Inspector General — List of Excluded Individuals/Entities
(LEIE)**, which OIG publishes as a CSV for loading into your own system:

- <https://oig.hhs.gov/exclusions/exclusions-list/>
- <https://oig.hhs.gov/exclusions/downloadables/UPDATED.csv>

US federal public domain. 83,782 exclusions as loaded — 80,355 individuals and
3,427 businesses.

## This names real people — read this before using it

A false positive here accuses somebody of healthcare fraud. The data makes that
easy to do by accident:

- **690 excluded people share the surname SMITH.** 591 share JOHNSON.
- **Four different excluded men are called some form of John Smith** — born
  1949 (California, internal medicine), 1951 (Colorado, counselor), 1960 (New
  York, plastic surgery) and 1970 (Pennsylvania, general practice).

So `check_provider_exclusion` **never returns a boolean.** It returns candidates
with `date_of_birth`, `state`, `city`, `specialty` and `npi`, and a
`match_confidence` describing what was actually matched:

| `match_confidence` | Meaning |
|---|---|
| `npi_match` | The NPI matched. An NPI identifies one provider — this is an identification. |
| `exact_name_only` | The full name matched exactly. **Not** an identification; several excluded people can share a name. |
| `strong_name_similarity` | Very close but not identical. A lead to check. |
| `weak_name_similarity` | Loosely similar. Most results at this level are different people. |

Pass `npi` whenever you have it — it is the difference between a lead and an
identification. Only **8,835 of 83,782** exclusions carry one, though, so for
most records date of birth is the practical discriminator.

## Currently excluded, not historically

Everyone on this list is currently excluded. The file has a reinstatement column
and it is empty on every row, because OIG **removes** a reinstated party rather
than flagging them. The ingest therefore deletes rows absent from each new file
— "no longer listed" is how a reinstatement reaches you, and there is no expired
state to read.

## What this does not cover

State Medicaid exclusion lists, SAM.gov debarment, and licensure actions that
have not led to an OIG exclusion. A clear result here is not a complete
background check, and `check_provider_exclusion` says so on every no-match
response.

## Refreshing

Monthly, when OIG republishes:

```bash
node scripts/ingest-leie.mjs
```

Re-runnable: rows upsert on a content hash, and anything absent from the new
file is deleted so reinstatements take effect. It refuses to load a file with
fewer than 50,000 rows rather than quietly emptying the table.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "leie": {
      "url": "https://gateway.pipeworx.io/leie/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/leie/mcp` returns the tools in the table
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

Both URLs reach the same gateway and the same 1576+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "leie": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-leie"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-leie
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Leie data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
