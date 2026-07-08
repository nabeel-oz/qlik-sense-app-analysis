# Qlik Sense App Analysis — Agent Skill

An [AI Skill](https://claude.com/skills) for exploring, analyzing,
visualizing, and governing Qlik Sense apps and Qlik Cloud analytics content
through the [Qlik MCP server](https://help.qlik.com/en-US/cloud-services/Subsystems/Hub/Content/Sense_Hub/QlikMCP/Qlik-MCP-server.htm).

Point Claude/ ChatGPT at a Qlik Cloud tenant with the Qlik MCP server connected, and
this skill teaches it how to find the right app, prefer governed
definitions over ad hoc ones, avoid Qlik's specific silent-failure traps,
and — when asked — build dashboards or author governed master items,
glossary terms, and data products.

It works out of the box against any tenant. It's designed to get better
the more you extend it with your own apps and domains — see
[Extending the skill](#extending-the-skill) below.

## Why this exists

Anthropic's [self-service analytics
write-up](https://claude.com/blog/how-anthropic-enables-self-service-data-analytics-with-claude)
frames most wrong answers a data agent gives as one of three failure modes:
**entity ambiguity** (which table/field is "revenue"?), **staleness** (is
this still true?), and **retrieval failure** (the right answer existed but
the agent never found it). Their fix against a SQL warehouse is a stack you
build by hand: canonical datasets, a semantic layer, lineage, and skills
that route the agent to a governed answer before it writes any raw SQL.

Qlik MCP tools address the same three failure modes — but several of the
layers the warehouse world has to build by hand already exist natively in
Qlik Cloud:

| Failure mode | Warehouse approach (build it yourself) | Qlik MCP approach (already there) |
|---|---|---|
| Entity ambiguity | Curate canonical dbt models, hope people don't duplicate them | **Data Products** (curated, documented datasets) and **Master Items** (governed dimensions/measures) — list before you build |
| Staleness | Freshness/completeness checks you write and maintain | `qlik_get_dataset_freshness`, `qlik_get_dataset_trust_score`, `qlik_get_dataset_quality_computation_status` are native tool calls |
| Retrieval failure | Distill a query corpus into reference docs by hand | `qlik_search`, `qlik_list_sheets`, `qlik_get_sheet_details` surface existing apps/charts directly — reuse before rebuilding |
| Lineage / "where did this come from" | Reconstruct the transformation graph from dbt manifests | `qlik_get_lineage` is a direct tool call (still one hop at a time — call it recursively) |
| Business glossary | An internal wiki or knowledge graph you pipe in yourself | A structured **Business Glossary** with a draft → verified → deprecated lifecycle, native to the platform |
| "Raw SQL" fallback | Hand-written SQL, reviewed adversarially before trusting it | `qlik_create_data_object` with a Qlik expression (set analysis) — Qlik performs the calculation, so the review is about *expression correctness*, not aggregation logic |

And two things are genuinely new in the Qlik MCP world, with no warehouse-skill
equivalent:

- **Authoring, not just querying.** The warehouse skill only reads. Qlik MCP
  tools can also *create* governed objects — master dimensions/measures,
  sheets, charts, data products, glossary terms — so this skill is part
  analyst, part steward, and every creation action carries a "does this
  already exist?" check.
- **Stateful selections.** SQL queries are stateless; Qlik selections persist
  across tool calls until cleared, and a selection on a value that doesn't
  exist **fails silently**. That's a class of silent failure SQL never has.

`qlik-sense-app-analysis/SKILL.md` lays out this mapping in full and turns it
into operating rules.

## Installing

The deployable skill is the `qlik-sense-app-analysis/` folder in this repo
(it contains `SKILL.md` and `references/`). Install it whichever way your
AI client loads skills:

- **Zip upload** (claude.ai / Claude Code skill upload): download
  `qlik-sense-app-analysis.zip` from the [Releases](../../releases) page, or
  rebuild it yourself — see [Building the zip](#building-the-zip).
- **Folder drop / plugin**: copy the `qlik-sense-app-analysis/` folder into
  your Claude client's skills directory, or add this repository as a plugin
  source.

Requires a Qlik MCP server connection with the tools listed in
`qlik-sense-app-analysis/references/qlik-mcp-tool-reference.md`. The skill
degrades gracefully if some tool categories aren't available (e.g. no Data
Products or Business Glossary licensed), but app discovery and analysis
tools are expected to be present.

## Extending the skill

The generic version works immediately, but for a real deployment you'll
usually want to point it at your own apps. Copy
`qlik-sense-app-analysis/references/app-profile-TEMPLATE.md` to
`references/<name>.md` per app or business domain and fill in the app ID,
the master items and data products already available, and any field-naming
quirks specific to that app. `SKILL.md` checks for a matching profile before
falling back to live discovery — no other changes required. This is
intentionally the same pattern Anthropic's article uses for per-domain
warehouse reference docs: narrow the search space with curated files before
the agent goes looking on its own.

## Structure

```
qlik-sense-app-analysis/                     — repository root
├── README.md
├── LICENSE
└── qlik-sense-app-analysis/                 — the skill (the deployable unit)
    ├── SKILL.md                             — the skill itself
    └── references/
        ├── qlik-mcp-tool-reference.md       — full tool catalogue by category
        ├── expression-gotchas.md            — Qlik expression / set analysis syntax rules
        ├── governance-workflows.md          — master item / glossary / data product lifecycle rules
        ├── visualization-guidelines.md      — chart-type selection, rendering fallbacks, dashboard/report pattern
        ├── design-rationale.md              — the reasoning behind the skill's structure
        └── app-profile-TEMPLATE.md          — copy per app/domain to extend the skill
```

The prebuilt `qlik-sense-app-analysis.zip` is published on the
[Releases](../../releases) page rather than committed to the repo. The
`dist/` folder is a local build artifact (git-ignored) produced by the
command below.

### Building the zip

From the repository root:

```bash
zip -r dist/qlik-sense-app-analysis.zip qlik-sense-app-analysis
```

or on Windows PowerShell:

```powershell
Compress-Archive -Path qlik-sense-app-analysis -DestinationPath dist/qlik-sense-app-analysis.zip -Force
```

The zip's top-level folder is `qlik-sense-app-analysis/` containing
`SKILL.md`, which is what skill uploaders expect.

## Contributing

Pull requests that add well-documented app or domain profiles, correct
tool behaviour as the Qlik MCP server evolves, or tighten up the
gotchas/governance references are welcome. Please keep customer-specific
detail (app IDs, internal field names) out of anything merged upstream —
those belong in a private fork's `references/` folder, not in the shared
template.

## License

MIT — see `LICENSE`.
