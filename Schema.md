# Format choice

Going with **markdown files plus YAML frontmatter, validated by a JSON Schema**. The reasoning is worth a paragraph because JSON's nulls are gross.

JSON forces you to choose between `null` (loud absence) and omitting the field (clean absence), and most schema-driven workflows end up demanding the loud version because the schema can't tell the difference. JSON5 fixes the syntax pain (comments, trailing commas) but inherits the same null situation. **Dhall** ("dhal") is genuinely the right shape for this problem: it has `Optional` as a first-class type, total functional composition for aggregation, imports for stitching files together, and no nulls anywhere; it would be my pick if you were the sole producer of the data. But every major LLM structured-output API speaks JSON Schema and none speak Dhall, so you'd burn the project on writing a JSON-to-Dhall converter. **JSON-LD** (which I'm guessing is what you meant by "jsond"; it's not a standard format I know) is built around URI-keyed cross-document references, which sounds perfect for "this shot relates to that look across projects," but the vocabulary overhead is heavy and you don't need full RDF semantics to point at a thing by id. **TOML** is unambiguous (no YAML Norway problem) but bad at nesting, which the lighting-fixture data demands. **Pure YAML** files would work but lose the diary affordance.

The cleanest split: JSON Schema is the **contract** the LLM fills against, but the **storage format** is markdown with YAML frontmatter, one entry per file. The LLM emits JSON matching the schema; a small script serializes that JSON as YAML at the top of a markdown file and templates a body below. The body is where "why we did it this way," "what I learned," and the years-later retrospective live, because those are prose, not fields. Concatenate the bodies chronologically across a project and you have the diary you described, with the structured frontmatter giving you queryability across projects. This also gives you Obsidian/Logseq/grep/rg/git for free, and markdown will still be readable in 50 years, which nothing else here can promise.

# Layout & ids

Each entry is a file. Cross-references are slug-based hierarchical ids, not UUIDs, because `ophelia-2025/scenes/12a/shots/3/takes/5` is greppable, human-readable, and survives a hard drive moving between machines. UUIDs are useful only as a rename-stability anchor and live as an optional `uuid:` field in frontmatter; queries use the slug.

```
production-log/
├── projects/
│   └── ophelia-2025/
│       ├── project.md                # the project envelope
│       ├── locations/
│       │   ├── diner-exterior.md
│       │   └── soundstage-a.md
│       ├── days/
│       │   └── 2025-08-14.md         # call sheet, weather, summary
│       ├── scenes/
│       │   └── 12a.md
│       ├── shots/
│       │   └── 12a-3.md
│       ├── takes/
│       │   └── 12a-3-t05.md          # often unnecessary; create only when notable
│       ├── looks/                    # the named lighting looks
│       │   ├── diner-noir.md
│       │   └── warm-window-pour.md
│       ├── setups/
│       │   └── diner-booth-day.md
│       ├── entries/                  # the chronological log
│       │   ├── 2025-08-14T07-30--prelight-diner.md
│       │   ├── 2025-08-14T19-42--setup-diner-booth.md
│       │   └── 2025-08-14T23-10--wrap-note.md
│       └── media/blobs/<sha256>.<ext>
├── catalog/                          # shared across projects
│   ├── fixtures/                     # one md per fixture model
│   ├── gels/
│   └── modifiers/
└── people/
    └── jane-doe-dp.md
```

The `entries/` folder is the time-ordered firehose. Everything in `looks/`, `setups/`, `scenes/`, etc. is a stable named entity that entries reference. Most entries reference a setup; a setup references a look; the look may have been defined on day one and reused for the rest of the shoot. This is exactly the structure that makes a post-mortem possible: you can ask "every entry that referenced the warm-window-pour look" and get a rolled-up history of one technique across the project.

# The schema

One JSON Schema, used for every kind of entry. The `entry_type` discriminator tells the LLM (and you) which sections are likely to be populated, but no section is required. Absence means absence; the schema does not allow nulls.

`schema/entry.schema.json`:

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://example.com/schemas/film-log/entry.json",
  "title": "Film Production Log Entry",
  "description": "Frontmatter for a single markdown entry. Filled by an LLM from voice memos, photos, and other captures. The LLM omits any field not grounded in source material; nulls are not allowed.",
  "type": "object",
  "additionalProperties": false,
  "required": ["id", "captured_at", "entry_type", "schema_version"],

  "properties": {
    "schema_version": {
      "type": "string",
      "description": "Semver of this schema. Lets you migrate later. Always set."
    },

    "id": {
      "type": "string",
      "pattern": "^[a-z0-9][a-z0-9-]*(/[a-z0-9][a-z0-9-]*)*$",
      "description": "Slug-based hierarchical id, e.g. 'ophelia-2025/entries/2025-08-14T19-42--setup-diner-booth'. Generated by the application, never invented by the LLM."
    },
    "uuid": {
      "type": "string",
      "format": "uuid",
      "description": "Optional rename-stability anchor."
    },

    "captured_at": {
      "type": "string",
      "format": "date-time",
      "description": "When the source memo or photo was captured (UTC). From file metadata when possible."
    },
    "captured_at_local": {
      "type": "string",
      "description": "Local wall-clock with offset, e.g. '2025-08-14T19:42:00-04:00'. Useful for sun position."
    },
    "logged_at": {
      "type": "string",
      "format": "date-time",
      "description": "When the entry was written or extracted. May differ from captured_at if extracted later."
    },

    "entry_type": {
      "type": "string",
      "enum": [
        "scout", "prelight_plan", "lighting_setup", "shot_record",
        "take_note", "exposure_reading", "color_chart",
        "issue", "lesson", "communication", "wrap_note",
        "look_definition", "retrospective", "general"
      ],
      "description": "What kind of situation this entry describes. Soft hint about which sections will be populated."
    },

    "tags": {
      "type": "array",
      "items": { "type": "string" },
      "description": "Free-form tags. Some conventions: 'safety', 'reshoot', 'signature-shot', 'lesson-learned'."
    },

    "refs": {
      "type": "object",
      "description": "References to other entities by slug id. Used to walk the graph during aggregation.",
      "additionalProperties": false,
      "properties": {
        "project":  { "type": "string", "description": "e.g. 'ophelia-2025'." },
        "day":      { "type": "string", "description": "e.g. 'ophelia-2025/days/2025-08-14'." },
        "location": { "type": "string", "description": "e.g. 'ophelia-2025/locations/diner-exterior'." },
        "scene":    { "type": "string", "description": "e.g. 'ophelia-2025/scenes/12a'." },
        "shot":     { "type": "string", "description": "e.g. 'ophelia-2025/shots/12a-3'." },
        "take":     { "type": "string", "description": "e.g. 'ophelia-2025/takes/12a-3-t05'." },
        "setup":    { "type": "string", "description": "e.g. 'ophelia-2025/setups/diner-booth-day'." },
        "look":     { "type": "string", "description": "Named look reference, e.g. 'ophelia-2025/looks/warm-window-pour'." },
        "people":   { "type": "array", "items": { "type": "string" }, "description": "e.g. ['people/jane-doe-dp']." },
        "related_entries": { "type": "array", "items": { "type": "string" }, "description": "Other entries this one builds on or contradicts." }
      }
    },

    "take": {
      "type": "object",
      "description": "Take-level info. Use for entry_type='take_note' or as a sidecar on shot_record.",
      "additionalProperties": false,
      "properties": {
        "number":    { "type": "integer", "minimum": 1 },
        "circled":   { "type": "boolean", "description": "Director/script supervisor selected this take." },
        "status":    { "type": "string", "enum": ["good", "ng", "incomplete", "false_start", "for_safety", "preferred"] },
        "duration_sec": { "type": "number" },
        "notes":     { "type": "string" }
      }
    },

    "look": {
      "type": "object",
      "description": "Used when entry_type='look_definition' or to record that this entry achieved/used a named look. The point of named looks is reuse and cross-project query.",
      "additionalProperties": false,
      "properties": {
        "nickname":     { "type": "string", "description": "Memorable handle, e.g. 'warm-window-pour', 'diner-noir', 'fluoro-from-hell'." },
        "one_line":     { "type": "string", "description": "One-sentence description that survives years of forgetting." },
        "intent":       { "type": "string", "description": "What this look is meant to evoke." },
        "key_recipe":   { "type": "string", "description": "Recipe in one paragraph: 'soft 12x12 of 216 from camera-left high, 1/4 CTO, negative fill camera-right, practicals dimmed to 30%'." },
        "originating_setup": { "type": "string", "description": "Slug id of the setup where this look first crystallized." },
        "exposure_anchor": {
          "type": "object",
          "additionalProperties": false,
          "properties": {
            "iso":        { "type": "integer" },
            "t_stop":     { "type": "number" },
            "fps":        { "type": "number" },
            "shutter_angle_deg": { "type": "number" }
          }
        },
        "white_balance_anchor": {
          "type": "object",
          "additionalProperties": false,
          "properties": {
            "cct_kelvin": { "type": "number" },
            "tint":       { "type": "number" }
          }
        }
      }
    },

    "scene": {
      "type": "object",
      "additionalProperties": false,
      "properties": {
        "number":   { "type": "string", "description": "e.g. '12A'." },
        "slug":     { "type": "string", "description": "e.g. 'INT. DINER - NIGHT'." },
        "story_time_of_day": {
          "type": "string",
          "enum": ["day","night","dawn","dusk","magic","interior_ambiguous","other"]
        },
        "synopsis": { "type": "string" }
      }
    },

    "shot": {
      "type": "object",
      "additionalProperties": false,
      "properties": {
        "slate":    { "type": "string", "description": "e.g. '12A-3'." },
        "size":     { "type": "string", "enum": ["ecu","xcu","cu","mcu","ms","mls","ls","xls","insert","cutaway","pov","ots","other"] },
        "movement": { "type": "string", "enum": ["static","handheld","dolly","slider","gimbal","crane","drone","car_mount","snorkel","other"] },
        "description": { "type": "string" },
        "selected_take": { "type": "integer", "description": "If known: the take that was printed/preferred." }
      }
    },

    "storyboard": {
      "type": "object",
      "description": "Storyboard reference, for the times you said 'we did it because the boards say so'.",
      "additionalProperties": false,
      "properties": {
        "page":     { "type": "integer" },
        "panel":    { "type": "string" },
        "revision": { "type": "string", "description": "e.g. 'rev-3', 'shooting'." },
        "media_id": { "type": "string", "description": "Hash id of the boarded panel image in media[]." },
        "deviation_note": { "type": "string", "description": "If we departed from the boards, why." }
      }
    },

    "location": {
      "type": "object",
      "additionalProperties": false,
      "properties": {
        "name":    { "type": "string" },
        "kind":    { "type": "string", "enum": ["interior_practical","exterior_street","soundstage","studio","nature","vehicle","other"] },
        "gps": {
          "type": "object",
          "additionalProperties": false,
          "required": ["latitude", "longitude"],
          "properties": {
            "latitude":  { "type": "number", "minimum": -90,  "maximum": 90 },
            "longitude": { "type": "number", "minimum": -180, "maximum": 180 },
            "altitude_m": { "type": "number" },
            "accuracy_m": { "type": "number" }
          }
        },
        "timezone_iana": { "type": "string" },
        "address_text":  { "type": "string", "description": "Free-text address; structure later if needed." },
        "hazards":       { "type": "array", "items": { "type": "string" } }
      }
    },

    "weather": {
      "type": "object",
      "additionalProperties": false,
      "properties": {
        "summary": { "type": "string" },
        "temperature_c": { "type": "number" },
        "wind": { "type": "string" },
        "sky":  { "type": "string" }
      }
    },

    "lighting_setup": {
      "type": "object",
      "additionalProperties": false,
      "properties": {
        "intent":              { "type": "string", "description": "What we were going for, in plain language." },
        "origin_description":  { "type": "string", "description": "What '0,0,0' means here. e.g. 'A-cam tripod foot, +Y toward subject'." },
        "ambient":             { "type": "string", "description": "Existing daylight, practicals, motivation." },
        "result":              { "type": "string", "description": "How it actually looked. Honest." },
        "fixtures":  { "type": "array", "items": { "$ref": "#/$defs/fixture_instance" } },
        "power":     { "$ref": "#/$defs/power_plan" }
      }
    },

    "exposure": {
      "type": "object",
      "additionalProperties": false,
      "properties": {
        "camera": {
          "type": "object",
          "additionalProperties": false,
          "properties": {
            "manufacturer": { "type": "string" },
            "model":        { "type": "string" },
            "sensor_mode":  { "type": "string" },
            "codec":        { "type": "string" }
          }
        },
        "lens": {
          "type": "object",
          "additionalProperties": false,
          "properties": {
            "manufacturer":    { "type": "string" },
            "series":          { "type": "string" },
            "focal_mm":        { "type": "number" },
            "t_stop_wide_open":{ "type": "number" },
            "focus_distance_ft":{ "type": "number" }
          }
        },
        "filtration": {
          "type": "object",
          "additionalProperties": false,
          "properties": {
            "nd_stops":   { "type": "number" },
            "ir_cut":     { "type": "boolean" },
            "polarizer":  { "type": "boolean" },
            "diffusion":  { "type": "string" },
            "other":      { "type": "array", "items": { "type": "string" } }
          }
        },
        "iso":               { "type": "integer" },
        "t_stop":            { "type": "number" },
        "fps":               { "type": "number" },
        "shutter_angle_deg": { "type": "number" },
        "metered_key_fc":    { "type": "number" },
        "metered_fill_fc":   { "type": "number" }
      }
    },

    "colorimetry": {
      "type": "object",
      "additionalProperties": false,
      "properties": {
        "white_balance": {
          "type": "object",
          "additionalProperties": false,
          "properties": {
            "cct_kelvin": { "type": "number" },
            "tint":       { "type": "number" },
            "source":     { "type": "string", "description": "'preset 5600', 'custom on Macbeth white', etc." }
          }
        },
        "color_chart_readings": {
          "type": "array",
          "items": {
            "type": "object",
            "additionalProperties": false,
            "properties": {
              "chart_type":   { "type": "string" },
              "media_id":     { "type": "string" },
              "under_lights": { "type": "string" },
              "notes":        { "type": "string" }
            }
          }
        },
        "look_lut": {
          "type": "object",
          "additionalProperties": false,
          "properties": {
            "name":     { "type": "string" },
            "version":  { "type": "string" },
            "from":     { "type": "string", "description": "DIT, post house, Livegrade, Resolve." },
            "media_id": { "type": "string" }
          }
        },
        "input_color_space":  { "type": "string" },
        "working_space":      { "type": "string" },
        "display_space":      { "type": "string" }
      }
    },

    "issue": {
      "type": "object",
      "additionalProperties": false,
      "properties": {
        "summary":    { "type": "string" },
        "detail":     { "type": "string" },
        "severity":   { "type": "string", "enum": ["low","medium","high","safety_critical"] },
        "status":     { "type": "string", "enum": ["open","mitigated","resolved","deferred"] },
        "resolution": { "type": "string" }
      }
    },

    "lesson": {
      "type": "object",
      "additionalProperties": false,
      "description": "The 'what I learned here' field. Promote to a Lesson when it's worth carrying forward.",
      "properties": {
        "subject":    { "type": "string" },
        "body":       { "type": "string" },
        "scope":      { "type": "string", "enum": ["one_off","recurring","rule"] }
      }
    },

    "rationale": {
      "type": "string",
      "description": "Why we did it this way. The single most valuable narrative field. Captures the decision behind the data."
    },

    "media": {
      "type": "array",
      "description": "All source files associated with this entry.",
      "items": {
        "type": "object",
        "additionalProperties": false,
        "required": ["media_id", "role"],
        "properties": {
          "media_id":      { "type": "string", "description": "sha256 prefix or content hash." },
          "role":          { "type": "string", "enum": [
            "voice_memo","scout_photo","setup_sketch","setup_bts_photo",
            "frame_grab","color_chart","document","floor_plan",
            "reference_image","video_clip","storyboard_panel","other"
          ]},
          "media_type":    { "type": "string", "description": "MIME, e.g. 'image/jpeg'." },
          "original_name": { "type": "string" },
          "captured_at":   { "type": "string", "format": "date-time" },
          "transcript":    { "type": "string", "description": "For audio: raw transcript before extraction." },
          "caption":       { "type": "string" }
        }
      }
    },

    "raw_text": {
      "type": "string",
      "description": "Source transcript or pasted note, preserved verbatim. When the schema evolves, you re-extract from this and your data heals itself."
    },

    "extraction": {
      "type": "object",
      "description": "How this entry was produced. Lets you re-run extraction or audit it.",
      "additionalProperties": false,
      "properties": {
        "model":          { "type": "string" },
        "extracted_at":   { "type": "string", "format": "date-time" },
        "confidence":     { "type": "number", "minimum": 0, "maximum": 1 },
        "low_confidence_fields":  { "type": "array", "items": { "type": "string" }, "description": "Dotted paths the LLM was unsure about." },
        "unresolved_questions":   { "type": "array", "items": { "type": "string" }, "description": "Follow-ups for the human." },
        "human_reviewed": { "type": "boolean" }
      }
    }
  },

  "$defs": {

    "fixture_instance": {
      "type": "object",
      "additionalProperties": false,
      "properties": {
        "nickname":          { "type": "string", "description": "'Key', 'Eyelight', 'Window blast'." },
        "manufacturer":      { "type": "string" },
        "model":             { "type": "string", "description": "'SkyPanel S60-C', 'Aputure 600d', 'Mole 12K'." },
        "kind":              { "type": "string", "enum": ["hmi","tungsten","led_bicolor","led_rgbw","led_rgbacl","led_full_spectrum","fluorescent","plasma","practical","space_light","other"] },
        "beam_shape":        { "type": "string", "enum": ["par","fresnel","soft","panel","tube","space_light","other"] },
        "max_draw_watts":    { "type": "number" },
        "intensity_percent": { "type": "number", "minimum": 0, "maximum": 100 },
        "cct_kelvin":        { "type": "number" },
        "tint_gm":           { "type": "number" },
        "hue_deg":           { "type": "number" },
        "saturation_percent":{ "type": "number" },
        "control": {
          "type": "object",
          "additionalProperties": false,
          "properties": {
            "protocol":      { "type": "string", "enum": ["dmx512","sacn","art_net","crmx","bluetooth","wifi","rdm","none"] },
            "universe":      { "type": "integer" },
            "start_channel": { "type": "integer" },
            "footprint":     { "type": "integer" },
            "profile":       { "type": "string" }
          }
        },
        "position": {
          "type": "object",
          "description": "Coordinates relative to the setup's origin_description.",
          "additionalProperties": false,
          "properties": {
            "x_m": { "type": "number", "description": "Right of origin." },
            "y_m": { "type": "number", "description": "Forward of origin." },
            "z_m": { "type": "number", "description": "Up from origin." }
          }
        },
        "aim": {
          "type": "object",
          "additionalProperties": false,
          "properties": {
            "yaw_deg":   { "type": "number" },
            "pitch_deg": { "type": "number" },
            "roll_deg":  { "type": "number" }
          }
        },
        "distance_to_subject_m": { "type": "number" },
        "rig": {
          "type": "object",
          "additionalProperties": false,
          "properties": {
            "kind":         { "type": "string", "enum": ["c_stand","combo","roller","menace","pipe","condor","scissor","car","drone","handheld","practical","other"] },
            "sandbags":     { "type": "integer" },
            "safety_cable": { "type": "boolean" },
            "operator":     { "type": "string" },
            "notes":        { "type": "string" }
          }
        },
        "gels": {
          "type": "array",
          "description": "Ordered fixture-side to subject-side.",
          "items": {
            "type": "object",
            "additionalProperties": false,
            "properties": {
              "manufacturer":  { "type": "string", "enum": ["rosco","lee","gam","apollo","other"] },
              "product_code":  { "type": "string", "description": "e.g. 'Rosco 3202', 'Lee 201'." },
              "display_name":  { "type": "string", "description": "e.g. 'Full CTB', '1/4 CTO', '216'." },
              "kind": { "type": "string", "enum": ["color_temperature_orange","color_temperature_blue","minus_green","plus_green","diffusion","color_effect","neutral_density","other"] }
            }
          }
        },
        "modifiers": {
          "type": "array",
          "items": {
            "type": "object",
            "additionalProperties": false,
            "properties": {
              "kind":   { "type": "string", "enum": ["softbox","china_ball","book_light","bounce","flag","net","cookie","silk","frame","egg_crate","blackout_drape","neg_fill","gobo_arm","snoot","barndoor","other"] },
              "name":   { "type": "string" },
              "size_ft":{ "type": "array", "items": { "type": "number" }, "minItems": 2, "maxItems": 2 },
              "notes":  { "type": "string" }
            }
          }
        },
        "operator": { "type": "string" },
        "notes":    { "type": "string" }
      }
    },

    "power_plan": {
      "type": "object",
      "additionalProperties": false,
      "properties": {
        "source":   { "type": "string", "enum": ["house_mains","generator","tie_in","battery","solar","other"] },
        "generator_kva":     { "type": "number" },
        "tie_in_location":   { "type": "string" },
        "legs": {
          "type": "array",
          "items": {
            "type": "object",
            "additionalProperties": false,
            "required": ["label"],
            "properties": {
              "label":          { "type": "string", "description": "'L1', 'L2', 'phase A'." },
              "amps_available": { "type": "number" },
              "est_amps_draw":  { "type": "number" },
              "fixture_nicknames": { "type": "array", "items": { "type": "string" } }
            }
          }
        },
        "notes": { "type": "string" }
      }
    }
  }
}
```

# The markdown entry shape

Every entry file looks like this. The frontmatter holds the JSON-Schema-validated data; the body holds the prose. The body sections are conventional (LLM is told to populate the relevant ones from the source memo) but free to omit; missing sections render as nothing.

`templates/entry.md`:

```markdown
---
schema_version: "0.1.0"
id: ophelia-2025/entries/2025-08-14T19-42--setup-diner-booth
captured_at: 2025-08-14T23:42:00Z
captured_at_local: "2025-08-14T19:42:00-04:00"
logged_at: 2025-08-15T03:10:00Z
entry_type: lighting_setup
tags: [low-key, single-source, signature-shot]

refs:
  project: ophelia-2025
  day:     ophelia-2025/days/2025-08-14
  location: ophelia-2025/locations/diner-exterior
  scene:   ophelia-2025/scenes/12a
  shot:    ophelia-2025/shots/12a-3
  setup:   ophelia-2025/setups/diner-booth-day
  look:    ophelia-2025/looks/diner-noir
  people:  [people/jane-doe-dp, people/marco-best-boy]

# (other top-level sections as needed: lighting_setup, exposure, colorimetry,
#  storyboard, weather, take, look, issue, lesson, rationale, media, raw_text, extraction)
---

## What we did

(Plain language description of the situation. The LLM fills this from the voice
memo, paraphrased into present tense.)

## Why we did it this way

(The rationale. Decisions, constraints, what the DP asked for, what we'd tried
that didn't work, why we landed here.)

## What I learned

(Lessons in prose. Promote to a structured `lesson:` field if it's worth
carrying forward.)

## Issues

(Bullets are fine here. Anything in the structured `issue` field gets
mentioned in prose too.)

## References

(Films, paintings, prior shots, photographer names. Mirror anything in
`refs.related_entries` here in prose.)

## Transcript

> (The raw audio memo transcript, preserved verbatim. Always keep this so
> you can re-extract when the schema evolves.)

## Retrospective

<!-- APPEND-ONLY. Add dated entries below; never edit the ones above. -->

(Empty at first. Years later you add notes here. See examples below.)
```

# Three examples

A lighting setup entry, fully filled, showing how structured + narrative coexist:

`projects/ophelia-2025/entries/2025-08-14T19-42--setup-diner-booth.md`:

```markdown
---
schema_version: "0.1.0"
id: ophelia-2025/entries/2025-08-14T19-42--setup-diner-booth
captured_at: 2025-08-14T23:42:00Z
captured_at_local: "2025-08-14T19:42:00-04:00"
logged_at: 2025-08-15T03:10:00Z
entry_type: lighting_setup
tags: [low-key, single-source, window-pour, signature-shot]

refs:
  project: ophelia-2025
  day:     ophelia-2025/days/2025-08-14
  location: ophelia-2025/locations/diner-exterior
  scene:   ophelia-2025/scenes/12a
  shot:    ophelia-2025/shots/12a-3
  setup:   ophelia-2025/setups/diner-booth-day
  look:    ophelia-2025/looks/diner-noir
  people:  [people/jane-doe-dp, people/marco-best-boy]

storyboard:
  page: 14
  panel: "B"
  revision: shooting
  deviation_note: "Boards have a wide; we are doing the medium first because the
    window light is going. Will pick up the wide tomorrow morning."

lighting_setup:
  intent: "Single soft source from camera-left through the diner window, hard
    negative fill on camera-right to crush the booth into noir."
  origin_description: "Tripod foot of A-cam, +Y toward the booth, +X camera-right."
  ambient: "Streetlight sodium spilling onto the sidewalk; we let it live and
    didn't gel it."
  result: "Beautiful at 1/2 stop under what we metered. The hot spot on the
    ketchup bottle made the frame; nobody planned that."

  fixtures:
    - nickname: "Key"
      manufacturer: ARRI
      model: "SkyPanel S60-C"
      kind: led_full_spectrum
      beam_shape: panel
      intensity_percent: 80
      cct_kelvin: 3200
      tint_gm: 2
      position: { x_m: -2.4, y_m: 1.8, z_m: 2.6 }
      aim: { yaw_deg: 70, pitch_deg: 20 }
      distance_to_subject_m: 3.1
      modifiers:
        - kind: frame
          name: "8x8 1/4 grid"
          size_ft: [8, 8]
      operator: marco-best-boy
      notes: "Boomed in through the window from outside on a combo-and-baby."

    - nickname: "Neg"
      manufacturer: Matthews
      model: "Floppy 4x4"
      kind: practical
      beam_shape: other
      modifiers:
        - kind: neg_fill
          name: "4x4 black floppy, camera-right"

    - nickname: "Practical-booth-pendant"
      manufacturer: ""
      model: "in-frame pendant lamp"
      kind: practical
      intensity_percent: 30
      notes: "Dimmed via a Lutron we rigged offscreen. Bulb is a 25W warm-white
        Edison."

  power:
    source: house_mains
    legs:
      - label: L1
        amps_available: 20
        est_amps_draw: 7
        fixture_nicknames: ["Key"]

exposure:
  camera:
    manufacturer: ARRI
    model: "Alexa Mini LF"
    sensor_mode: "Open Gate 4.5K"
    codec: ARRIRAW
  lens:
    manufacturer: Cooke
    series: "S4/i"
    focal_mm: 32
    t_stop_wide_open: 2.0
  iso: 800
  t_stop: 2.4
  fps: 24
  shutter_angle_deg: 172.8
  metered_key_fc: 28
  metered_fill_fc: 2

colorimetry:
  white_balance:
    cct_kelvin: 3200
    tint: 4
    source: "custom on white card under key only"
  look_lut:
    name: "Ophelia Show LUT"
    version: "v3"
    from: "Pomfort Livegrade Pro"
  input_color_space: "ARRI LogC4 / AWG4"
  display_space: "Rec.709 Gamma 2.4"

rationale: |
  Jane wanted the booth to feel abandoned even though the actor is in it.
  We tried a two-source key/fill on the prelight and it looked like a
  sitcom. Killing the fill and letting the practical do all the work on
  camera-right gave us the loneliness she was after. The 8x8 frame outside
  the window is doing the heavy lifting; the SkyPanel by itself was too
  contrasty, the grid softened the falloff just enough.

lesson:
  subject: "Trust the practical when the script wants loneliness"
  body: |
    Every time we've tried fill on a 'lonely diner' beat it kills the mood.
    Practical at 30% + neg fill on the camera side has now worked three
    times in a row. Promoting this to a rule.
  scope: rule

media:
  - media_id: "8c4f1d9a"
    role: voice_memo
    media_type: audio/m4a
    captured_at: 2025-08-14T23:42:00Z
    transcript: |
      (preserved verbatim, 4 minutes 12 seconds)
  - media_id: "a13b9e02"
    role: setup_bts_photo
    media_type: image/jpeg
    captured_at: 2025-08-14T23:50:00Z
  - media_id: "ff20c811"
    role: frame_grab
    media_type: image/png
    captured_at: 2025-08-15T00:03:00Z
    caption: "Final monitor grab of take 4."

extraction:
  model: "claude-opus-4-7"
  extracted_at: 2025-08-15T03:10:00Z
  confidence: 0.78
  low_confidence_fields:
    - "lighting_setup.fixtures[0].position.x_m"
    - "exposure.metered_fill_fc"
  unresolved_questions:
    - "Was the practical bulb actually 25W or did Marco swap it for 40W?"
  human_reviewed: false
---

## What we did

Single-source key through the diner window via an 8x8 1/4-grid frame, SkyPanel
boomed in from outside on a combo. No fill from the camera side; let a 4x4
black floppy crush the booth into shadow. Practical pendant in frame dimmed to
roughly thirty percent. Sodium streetlight outside left ungeled.

## Why we did it this way

See `rationale` in frontmatter. Short version: the prelight version with fill
felt like a sitcom; killing the fill and dimming the practical bought us the
loneliness Jane wanted.

## What I learned

Promoted to a rule, see `lesson`. Practical + neg fill is now my default for
"lonely person in a public space" beats. Stop second-guessing it.

## Issues

- The combo holding the SkyPanel was on a sidewalk grate. We sandbagged
  three deep but I want a better solution next time we shoot through a window.
- Marco's headset battery died in the third hour. Spares from now on.

## References

- *Drive* (Sigel, 2011) — the parking lot scenes for the sodium spill discipline.
- Edward Hopper, *Nighthawks* — referenced explicitly by Jane during the prelight.

## Transcript

> Okay it's seven forty-two, we just locked the booth setup for twelve A...
> (full transcript continues)

## Retrospective

<!-- APPEND-ONLY -->
```

A look definition entry, which other entries reference by slug:

`projects/ophelia-2025/looks/diner-noir.md`:

```markdown
---
schema_version: "0.1.0"
id: ophelia-2025/looks/diner-noir
captured_at: 2025-08-14T23:42:00Z
entry_type: look_definition
tags: [look, low-key, signature]

refs:
  project: ophelia-2025

look:
  nickname: diner-noir
  one_line: "Single warm window source, hard neg fill on the camera side, dimmed practical."
  intent: "Loneliness in a public space. The character is not alone in the room
    but is alone in the universe."
  key_recipe: |
    Soft warm key (SkyPanel through 8x8 1/4 grid) from outside the window
    at ~70 degrees off-axis from camera. Camera-side neg fill (4x4 floppy
    minimum, 8x8 if room allows). In-frame practical at 25-30%. Let any
    sodium streetlight live.
  originating_setup: ophelia-2025/setups/diner-booth-day
  exposure_anchor:
    iso: 800
    t_stop: 2.4
    fps: 24
    shutter_angle_deg: 172.8
  white_balance_anchor:
    cct_kelvin: 3200
    tint: 4
---

## What this look is for

Defined on the diner booth setup, day 4 of *Ophelia*. The point of naming it
is that we'll use it again for the bus stop scene (sc 28) and the apartment
window scene (sc 41), and I want the same recipe both times.

## Why it works

The key is doing two jobs at once: it's the window light AND the only soft
source in the frame. Anything fighting it on the camera side reads as a
production light, which kills the loneliness. The neg fill is what makes
this look "noir" rather than just "low-key" — it gives you the hard edge
on the dark side of the face that the dimmed practical can then re-soften
asymmetrically.

## When NOT to use it

- Anything wider than a medium falls apart; the practical can't carry the
  background.
- Daytime exteriors. Obviously.
- Comedy beats — the negative fill reads as cruel.

## Retrospective

<!-- APPEND-ONLY -->
```

A retrospective addition, showing how a years-later note slots in. The append-only convention is what makes this safe; you never go back and edit what you said at the time, you append a new dated section that reframes it:

```markdown
## Retrospective

<!-- APPEND-ONLY -->

### 2027-03-14 — *Ophelia* premiered at SXSW

The diner-noir look is the one shot from the trailer everyone keeps asking
about. Worth knowing, three years on, that the thing that sold it wasn't
any of the stuff in the rationale. It was the ketchup bottle. The hot spot
I called out as "nobody planned that" is what the audience locks onto.

Lesson here is more about my own taste than about the lighting: the
designed parts of the frame are the floor; the accidents are what the
audience remembers. I should leave more room for accidents in my plots.

### 2028-09-02 — Used this look on the *Halcyon* commercial pitch

Jane asked for "diner-noir but for a luxury watch ad." It did NOT translate.
The neg fill that reads as loneliness in a narrative beat reads as cheap in
a commercial; we ended up doing the inverse, fill on the camera side, dim
the key. Note for the rule: this look is genre-locked to drama. Don't ship
it to luxury work without expecting to reverse it.
```

# Aggregation toward a post-mortem

Because every entry is a self-contained file with structured frontmatter, the post-mortem is a script, not a feature. Walk `projects/<id>/entries/`, parse frontmatter, group by various keys, render markdown. Some queries you'd run on day one:

The diary itself is `cat $(ls projects/ophelia-2025/entries/*.md | sort) | render-bodies` — concatenate every entry's body in chronological order, render the H2 sections under per-day chapter headers, and you have a chronological narrative the LLM can summarize, but more importantly that you can read.

For the post-mortem: group entries by `refs.look`, render the originating look definition, then list every entry that referenced it with its rationale and result, then concatenate the retrospective sections. You get a complete biography of one technique on one project.

For cross-project queries: same thing across all projects, grouped by `tags` or by `look.nickname` (you can name looks consistently across projects on purpose: a `warm-window-pour` from 2025 and a `warm-window-pour` from 2028 are explicitly the same recipe and you want to be able to ask "did it work better on tungsten cameras or on Alexa LF?").

A starter aggregation script in any language reads frontmatter with a YAML library, filters by selectors over `refs`/`tags`/`entry_type`, and templates markdown. Five hundred lines of Python or Go gets you 90% of the post-mortem story. The schema being JSON-validatable means you can also push the whole corpus into SQLite with one column per entry as JSONB and run real queries when prose-rendering isn't enough.

# LLM extraction flow

The contract: the LLM is given (a) the schema, (b) the source memo transcript and any extracted EXIF/GPS, (c) any context about the current project's existing entities (slug ids of locations, scenes, looks, setups already defined). It produces JSON validated against the schema with structured-output mode. A post-processor turns that JSON into a markdown file with the YAML serialized as frontmatter and the prose body templated from any narrative-shaped fields (the LLM is asked to produce both the structured fields *and* a `_body_sections` map keyed by section name; the post-processor strips that key and writes it as the body).

The extraction prompt should explicitly say: omit fields not grounded in the source; populate `extraction.low_confidence_fields` with dotted paths for anything you guessed; populate `unresolved_questions` for anything a follow-up call to the user would clarify; never invent slug ids, only use ids that appear in the context block. That last rule is the one that goes wrong without nagging.

`raw_text` is preserved on every entry. When the schema evolves, you re-run extraction over the corpus's transcripts and the data heals itself. Without that field, every schema change is destructive, which is the failure mode that kills systems like this.

# One caveat

YAML's edge cases: the Norway problem (`country: NO` parses as boolean false in YAML 1.1), version strings becoming floats (`version: 3.10` becomes 3.1), and quoted vs unquoted ambiguity. The mitigation is "quote any string that could be misparsed," and the LLM should be told to quote conservatively in the extraction prompt. If this bites enough, switch the frontmatter to TOML; the schema doesn't change, only the serializer does.