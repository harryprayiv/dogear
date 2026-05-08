# dogear

Dogear is a schema and an LLM extraction contract for logging what happens on a film set. It is built for a chief lighting technician who wants notes that survive across years and projects, rather than dying in a phone's voice-memo app three months after wrap.

The name is the kind of fold you put in a book page to mark a spot you might come back to. An entry in dogear marks a place worth returning to: a setup that worked, a mistake worth remembering, a look you want to use again on the next project.

## What this repository contains

This is not an application. It is the foundation an application gets built on:

- A JSON Schema describing the shape of one log entry.
- A wrapper schema for LLM structured-output calls.
- A system prompt that instructs an LLM to extract entries from voice memos with deterministic, field-consistent results.
- A folder convention for how entries, locations, scenes, shots, setups, and named looks live on disk.
- Templates for the markdown body sections that make the structured data legible to a human.

The application that runs the extraction loop, syncs media, and renders aggregations sits downstream of this contract. Those layers are easy to swap. The schema is hard to change once a year of data exists, so the schema is what this repository nails down first.

## The problem

Every time a chief lighting technician finishes a project, the things they actually learned, which fixture worked at which distance with which gel, why the DP wanted that one specific feeling, what they told themselves they would do differently next time, live in some mix of voice memos, sketchbooks, paper notebooks, and memory. Six months later most of it is gone. Six years later it is all gone except the war stories.

A working log has to fit into on-set workflow without slowing it down, which means voice memos and photos, not forms. It also has to be queryable years later, which means structured fields, not just prose. The standard solution is "I will write it up properly later," and that is a lie.

Dogear's bet is that the first capture stays unstructured (a voice memo, a photo, a typed line) and that an LLM, constrained by a schema, fills in the structure consistently after the fact. The data is correct because the schema enforces shape. The data is honest because the LLM is instructed to omit anything not grounded in the source rather than guess.

## How it works

A capture happens on set: a voice memo, a photo with EXIF, a typed note. A small ingestion step transcribes audio, pulls EXIF and GPS, and assembles a packet that contains the source material plus the current project's slug ids (locations, scenes, shots, setups, and looks already defined). That packet goes to an LLM with the dogear extraction prompt and the entry schema as the structured-output contract.

The LLM returns a JSON object with three parts: a `frontmatter` block of structured fields validated against the schema, a `body_sections` map of markdown prose for the human-readable portion of the entry, and a `suggested_new_entities` list of slug ids the LLM thinks should be created (a new setup, a new named look, a new location). A post-processor writes the entry as a markdown file with YAML frontmatter, copies the source media into the project's blob store, and surfaces the suggested new entities for human review.

What lives on disk is plain markdown. Years from now, when whatever app sits on top of this is gone or broken, the files are still readable in any text editor, still greppable, still indexable by any tool you want to throw at them.

## Repository layout

```
dogear/
├── schema/
│   ├── entry.schema.json       # canonical entry schema
│   └── wrapper.schema.json     # the {frontmatter, body_sections, suggested_new_entities} envelope
├── prompts/
│   └── extraction.md           # the system prompt used for LLM extraction
├── examples/
│   ├── lighting-setup.md       # a fully populated lighting_setup entry
│   ├── look-definition.md      # a look_definition entry for a named look
│   └── retrospective-append.md # showing how years-later notes get appended
├── templates/
│   └── entry.md                # the empty template body
└── README.md
```

A user's actual log lives outside this repo, in a folder structure like:

```
~/film-log/
├── projects/
│   └── ophelia-2025/
│       ├── project.md
│       ├── locations/
│       ├── days/
│       ├── scenes/
│       ├── shots/
│       ├── takes/
│       ├── setups/
│       ├── looks/
│       ├── entries/             # the chronological log; new captures land here
│       └── media/blobs/
├── catalog/                     # reused across projects
│   ├── fixtures/
│   ├── gels/
│   └── modifiers/
└── people/
```

## The entry schema

`schema/entry.schema.json`. Every log entry has a slug-based hierarchical id, a captured-at timestamp, an entry type, and a schema version. Beyond that, every section is optional. An entry might populate only a few sections (a thirty-second voice memo about a problem with a sandbag) or many (a complete lighting setup with exposure, colorimetry, fixtures, gels, and rationale). Absent fields are omitted from the output, never set to null.

Cross-references between entries use slug ids: `ophelia-2025/scenes/12a`, `ophelia-2025/looks/diner-noir`, `people/jane-doe-dp`. The slug is greppable, narrative, and stable enough to live in a diary. An optional UUID provides rename stability when needed.

The notable sections:

- `lighting_setup` describes one arrangement of fixtures, with each fixture's position, aim, gels, modifiers, rig, and operator captured inline.
- `exposure` and `colorimetry` capture camera state and color pipeline at the time the setup was struck.
- `look` defines or references a named lighting look (for example `diner-noir`) so the same recipe can be reused across scenes and projects.
- `take` records the result of a specific take (good, ng, circled by the script supervisor, etc.).
- `storyboard` links to the boarded panel and notes any deliberate departure from the boards.
- `lesson` and `rationale` are the narrative anchors that make this a diary years later, not only a database.
- `extraction` records how the entry was produced (model, confidence, low-confidence fields, unresolved questions) so a human can audit or re-run extraction later.

A walkthrough of every field lives in `schema/entry.schema.json` itself; the field descriptions are written for a human reader and double as documentation.

## The extraction prompt

`prompts/extraction.md` defines the contract for the LLM. The non-negotiable rules:

1. Only what is grounded in the source.
2. Omit, never null.
3. Never invent slug ids; only use ids that appear in the project context block.
4. Preserve the source verbatim in `raw_text` so re-extraction works after schema changes.
5. Report uncertainty in `extraction.low_confidence_fields` and `extraction.unresolved_questions`.
6. Do not reframe, soften, or polish. Match the user's tense and register.

The prompt expects a structured-output API call (Anthropic tool use, OpenAI `response_format`, Google structured output). The wrapper schema in `schema/wrapper.schema.json` is what gets passed as the response format; the entry schema is referenced by `$ref` from inside it.

A one-shot example pair (input voice memo plus expected output JSON) is included in the prompt file. Send it as a preceding user/assistant turn before the real extraction call. Behavior is meaningfully more consistent with the example than without.

Use temperature between 0.1 and 0.3. Higher and the model starts inventing fixture model variants and gel codes that were never said.

## Quick start

This assumes you have an LLM API key, a transcription tool, and a folder you want to start logging into.

1. Clone this repo. Read `prompts/extraction.md` and `examples/lighting-setup.md` to understand what the output looks like.
2. Initialize a log folder following the layout under `~/film-log/projects/<your-project>/`. The minimum needed for the first entry is `entries/` and `project.md`.
3. Define the project itself: write `project.md` with a slug id (for example `ophelia-2025`), the title, and any people, locations, or scenes already known. These become the `project_context` block for extraction.
4. Record a voice memo on set. Save the audio somewhere your ingestion script can find it.
5. Transcribe the audio. Whisper, MacWhisper, or whatever you prefer.
6. Build the LLM call: system prompt is `prompts/extraction.md`, response format is `schema/wrapper.schema.json`, user message is the project context plus the file metadata plus the transcript, formatted as the prompt expects.
7. Receive the wrapper JSON. Validate. Write the markdown file to `entries/<timestamp>--<slug>.md` with the frontmatter on top and the body sections rendered below.
8. Review `suggested_new_entities`. Create the new setup or look files if they make sense.

The application that automates steps four through eight is not in this repo. The first version is fifty lines of any language you like. Build the dumbest possible version first. The schema is what carries the weight.

## Aggregation

Because every entry is a self-contained markdown file with structured frontmatter, queries are file operations:

- The chronological project diary is `cat $(ls projects/<id>/entries/*.md | sort)` piped through a small renderer that strips frontmatter and renders body sections under per-day chapter headers.
- The post-mortem of a single named look is every entry where `refs.look` matches that look's slug, sorted by `captured_at`, with the originating look definition rendered first.
- Cross-project queries (every time you used `warm-window-pour` across all projects) work the same way, one folder up.
- For real querying, load the corpus into SQLite with one row per entry and a JSON column for the frontmatter. Most useful queries become two-line SQL.

The "production diary" use case is the test of whether the system is working: at the end of a project, can you produce a chronological prose narrative of the shoot, with structured details available where useful, that you would actually want to read in five years? If the schema and the prompt are working, yes. If not, the schema is wrong; fix it and re-extract from `raw_text`.

## Design principles

The schema is a maximum, not a minimum. A thirty-second voice memo produces a thirty-second entry.

Absence is meaningful. Omitted fields mean "not stated"; nulls are not used.

Slug ids over UUIDs for everything a human reads. UUIDs only as a rename anchor where stability matters.

Source preserved on every entry. When the schema changes, you re-extract and your data heals itself. The day you delete `raw_text` to save space is the day the archive starts rotting.

The structured fields are for queries. The prose body is for the diary. Both have to work for the system to be worth using. A schema that produces queryable garbage is no better than a notebook that produces unqueryable poetry.

The retrospective section in each entry is append-only, dated, never edited. Past-you said what past-you said. Future-you adds a new dated note that comments on it. This is how the archive stays honest with itself.

## Status

What exists in this repo:

- The entry schema (`schema/entry.schema.json`).
- The wrapper schema for LLM calls (`schema/wrapper.schema.json`).
- The extraction system prompt (`prompts/extraction.md`).
- One worked example for each of: lighting setup, look definition, retrospective append.
- This README.

What does not exist yet:

- An ingestion CLI (transcribe, extract, write).
- An aggregation CLI (diary, post-mortem, named-look biographies).
- A capture mobile app.
- Any kind of search UI.

None of those are blockers. The schema and the prompt are usable as-is with any LLM API client and a few lines of glue.

## License

MIT. See `LICENSE`.

## Contributing

Schema changes are the careful part. Bump `schema_version`, write a migration note, and re-extract from `raw_text` across the test corpus before merging. Adding a new optional field is cheap. Removing or renaming a field is an event.

Prompt changes need before-and-after extractions on the example corpus to confirm behavior did not regress. Subtle prompt edits change extraction style in ways that only show up across many entries.

Examples should be drawn from real situations rather than invented ones. Invented examples teach the LLM to invent.