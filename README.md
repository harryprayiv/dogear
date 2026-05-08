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