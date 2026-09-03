# IKE Topic Registry Standards

## Purpose

The `topic-registry.yaml` file is the catalog of all topics in a topic library module. It is
**generated from the topic fragments**, not authored: the fragments are the source of every
value in it, and it is rebuilt from them rather than edited. See § "The registry is derived,
not authored."

It serves three functions:

1. **Corpus view**: A single file giving the whole corpus map — what exists, its status, and
   how it is assembled — without opening several hundred fragments.
2. **Assembly planning**: Authors and tooling use the registry to understand what content
   exists, its status, and its dependencies when constructing assembly documents.
3. **Claude navigation**: The registry provides Claude (chat or Claude Code) with a searchable
   index of the corpus so that content can be located by keyword, topic-id, or domain without
   uploading the full topic library.

## File Location

```
{topic-library-module}/src/docs/asciidoc/topic-registry.yaml
```

The registry travels with the topic content it catalogs. When the topic library is packaged
and unpacked into a dependent module's `target/` directory, the registry is available alongside
the topics.

## Schema

```yaml
# topic-registry.yaml
registry-version: "1.1"            # schema version for forward compatibility
generated: 2025-06-15              # ISO date of last registry update
topic-count: 247                   # total number of topic entries (validation target)

domains:
  - id: arch                       # domain identifier, used as topic-id prefix
    title: "System Architecture"   # human-readable domain name
    description: >                 # optional: domain scope statement
      Topics covering system architecture, design patterns,
      and infrastructure decisions.
    topics:                        # ordered list of topics in this domain
      - id: arch-overview
        file: topics/architecture/overview.adoc
        title: "Architecture Overview"
        type: concept
        keywords: [architecture, overview, IKE, layers]
        status: published
        dependencies: []
        related: []
        summary: >
          High-level overview of the IKE layered architecture including
          knowledge graph, reasoning, and API tiers. Covers the separation
          between storage, inference, and presentation layers.

      - id: arch-dl-classifier
        file: topics/architecture/dl-classifier.adoc
        title: "Classifier Architecture"
        type: concept
        keywords: [classifier, reasoning, EL++, inference]
        status: published
        dependencies: [arch-overview]
        related: [term-dl-axioms]  # covers similar ground from architecture angle
        summary: >
          Describes the classifier subsystem architecture including integration
          points, performance characteristics, and the EL++ profile constraints
          that enable polynomial-time reasoning.

assemblies:                        # catalog of assembly documents
  - id: compendium
    file: compendium.adoc
    title: "IKE Compendium"
    description: "Master assembly containing all topics."
    sections:                      # hierarchical structure mirroring the assembly
      - heading: "Architecture"
        sections:
          - heading: "Core Patterns"
            topic-refs: [arch-overview, arch-coord-versioning, arch-module-coordinates]
          - heading: "Reasoning"
            topic-refs: [arch-dl-classifier, arch-inference-pipeline]
      - heading: "Terminology Management"
        sections:
          - heading: "SNOMED CT"
            topic-refs: [term-snomed-concept-model, term-dl-axioms]
          - heading: "LOINC"
            topic-refs: [term-loinc-part-mapping]

  - id: versioning-guide
    file: guides/versioning-guide.adoc
    title: "Versioning Guide"
    description: "Targeted guide for version management."
    sections:
      - heading: "Core Concepts"
        topic-refs: [arch-coord-versioning, arch-module-coordinates]
      - heading: "Procedures"
        topic-refs: [ops-version-migration, ops-version-conflict-resolution]
      - heading: "Reference"
        topic-refs: [ref-coordinate-fields]
```

## The registry is derived, not authored

**No field of a topic entry is written by hand.** Every one is either read from the fragment's
attribute block or computed from the fragment's content, and the registry is regenerated from
the corpus rather than edited. The fragment is the authored source; the registry is a
projection of it.

This is what lets a fragment be located, catalogued and indexed from the file alone even when
the registry is missing, stale or wrong. A registry that disagrees with the corpus is not a
discrepancy to adjudicate — it is out of date, and regenerating it is the fix.

### Where each topic field comes from

| Field          | Source                                                                    |
|----------------|---------------------------------------------------------------------------|
| `id`           | The fragment's literal anchor, `[[arch-coord-versioning]]`.               |
| `file`         | The fragment's path, relative to the `src/docs/asciidoc/` root.          |
| `title`        | The fragment's level-1 heading.                                          |
| `domain`       | The topic id's prefix.                                                    |
| `type`         | `:topic-type:`                                                            |
| `status`       | `:topic-status:`                                                          |
| `keywords`     | `:topic-keywords:`, split on commas.                                     |
| `summary`      | `:topic-summary:`, or the fragment's opening paragraph when no attribute is present. |
| `related`      | `:topic-related:`, split on commas.                                      |
| `supersedes`   | `:topic-supersedes:`                                                      |
| `notes`        | `:topic-notes:`                                                           |
| `dependencies` | The `xref:` targets in the fragment body, in document order, excluding self-references. The field is *defined* as the topic's cross-references, so it is read from them rather than restated. |

A topic file is any `.adoc` carrying a literal anchor immediately followed by a level-1
heading. This is the discriminator rather than a directory convention: topics and assemblies
routinely sit in the same directory, and an assembly has no such anchor.

Topic order within a domain is reading order — first appearance across the assemblies' include
sequences — not filename order. Topics belonging to no assembly sort last, by path.

### `char-count` is not a registry field

It was removed. It cached a number `wc` computes in a millisecond, it was stale the moment
anyone edited the topic, and the standard never defined it precisely enough to reproduce.
Granularity bounds are checked by measuring the file at the time of the check, which is both
simpler and more accurate than consulting a copy recorded at the last decomposition pass.

### Values that no fragment can supply

Four values describe *collections* rather than any single topic, so no fragment states them.
They are authored in a small side file beside the registry, `topic-registry-meta.yaml`, and
merged in during generation:

```yaml
registry-version: "1.1"
domains:
  arch:
    title: "System Architecture"
    description: >
      Topics covering system architecture, design patterns, and
      infrastructure decisions.
assemblies:
  versioning-guide:
    description: "Targeted guide for version management."
```

This file scales with the number of domains and assemblies, not with the number of topics.
`generated` is the generation date and `topic-count` is the count of entries; both are
computed.

## Field Definitions: Assembly Entry

Like topic entries, assembly entries are generated. An assembly is any `.adoc` that carries no
topic anchor and `include::`s at least one topic file.

| Field          | Source                                                                   |
|----------------|--------------------------------------------------------------------------|
| `id`           | The assembly file's basename.                                            |
| `file`         | The assembly file's path, relative to the `src/docs/asciidoc/` root.    |
| `title`        | The assembly document's level-1 heading.                                 |
| `description`  | `topic-registry-meta.yaml` — the one assembly value nothing derives.    |
| `sections`     | The assembly document's own heading structure and `include::` order.     |

### Assembly Section Structure

Assembly entries use nested `sections` to capture the heading hierarchy of the assembled
document. This gives Claude and authors structural context — not just which topics are
included, but where they sit in the document hierarchy.

**Sections are read from the assembly document, never authored into the registry.** Each
heading becomes a section; each `include::` of a topic file contributes that topic's id to the
enclosing section's `topic-refs`, in document order. Headings that include no topics are
omitted.

This closes a gap that no validation rule could: when `sections` was authored separately, a
registry whose structure had drifted from the assembly it described passed every check, because
every check compared the registry against itself. Reading the structure from the document makes
the drift unrepresentable.

| Field          | Type       | Description                                                  |
|----------------|------------|--------------------------------------------------------------|
| `heading`      | string     | The section heading text as it appears in the assembly.      |
| `topic-refs`   | string[]   | Ordered list of `topic-id` values included under this heading. |
| `sections`     | section[]  | Optional nested subsections.                                 |

Sections may nest to match the assembly's heading depth. The `topic-refs` at each level
list the topics included directly under that heading, in document order. A section may have
both `topic-refs` and child `sections` if it contains both directly included topics and
subsections.

## Topic ID Construction Rules

1. Format: `{domain-prefix}-{descriptive-slug}`
2. Domain prefix: 2–5 lowercase characters matching a `domains[].id` in the registry.
3. Slug: lowercase kebab-case, 2–5 words, descriptive of content.
4. Total length: aim for under 40 characters.
5. **Immutability**: Once a `topic-id` is assigned and committed, it must not be changed. Other
   topics, assemblies, and external documents may reference it. If a topic's scope changes
   substantially, create a new topic and set `status: deprecated` on the old one with a
   `notes` field pointing to the replacement.

Examples:
- `arch-coord-versioning` — architecture domain, describes coordinate-based versioning
- `term-snomed-concept-model` — terminology domain, SNOMED CT concept model
- `safe-usc-hazard-analysis` — safety domain, unsafe control action hazard analysis
- `ops-maven-release-process` — operations domain, Maven release procedure

## Status Lifecycle

```
draft → review → published
                     ↓
                deprecated

draft → proposed → review        (proposal adopted)
        proposed → deprecated    (proposal declined or superseded)
```

- **draft**: Content is being authored or decomposed. May contain TODOs and placeholders.
- **proposed**: Content-complete design proposal awaiting an adoption decision. Distinct from
  `draft` (content still being authored): a proposed topic is ready to read, but the approach
  it argues for has not been decided. On adoption, move to `review`; if declined or
  superseded, move to `deprecated`.
- **review**: Content is complete and awaiting technical review.
- **published**: Content is reviewed and approved for inclusion in assemblies.
- **deprecated**: Content is superseded or no longer applicable. Retained in the registry for
  reference stability but excluded from new assemblies. Set `supersedes` on the replacement
  topic if one exists.

## Keyword Guidelines

Keywords are the primary mechanism for Claude to locate topics by subject matter. Follow these
rules:

1. **3–8 keywords per topic.** Fewer is too sparse for search; more dilutes relevance.
2. **Include synonyms and abbreviations**: If the topic discusses "description logic," also
   include `DL` and `classifier`. If it covers SNOMED CT, include `SCT`.
3. **Do not repeat title words**: The title is already searchable. Keywords should expand
   coverage.
4. **Prefer specific terms over generic**: `stamp-coordinate` over `coordinate`;
   `el-profile` over `profile`.
5. **Include the names of key standards, systems, or specifications** referenced in the topic.

## Summary Guidelines

Summaries serve triple duty: human-readable abstracts, Claude search targets, and redundancy
detection signals. They are the primary mechanism by which Claude identifies content overlap
across sessions. Invest effort in making them specific and term-rich.

1. Be 1–3 sentences, 150–400 characters. This is longer than a typical abstract — the extra
   space is needed for the technical terms that drive redundancy detection.
2. Use indicative mood: "Describes the coordinate-based versioning pattern..." not "This topic
   describes..."
3. Include 3–5 key technical terms not already in `keywords` or `title`. Prioritize terms
   that would help identify overlap with other topics — the specific standards, formalisms,
   patterns, and domain concepts discussed in the body.
4. Mention the *angle* or *perspective* of the topic when relevant: "from the terminology
   authoring perspective" or "focusing on build-time validation." This helps distinguish
   intentionally overlapping topics.
5. Be specific enough that a reader (or Claude) can determine relevance and potential overlap
   without opening the file.

Bad: "Covers versioning." (too vague, no technical terms, useless for redundancy detection)

Bad: "Describes coordinate-based versioning." (marginally better but still lacks the specific
terms that would trigger overlap detection)

Good: "Describes the coordinate-based versioning pattern where each component version is
identified by module, path, and temporal coordinates within the STAMP model. Covers the
relationship between coordinates and the version graph used for dependency resolution."

## Maintenance Rules

### When to Regenerate

Regenerate the registry whenever the corpus changes: a topic created, modified, split, merged
or deprecated; a topic's attributes edited; an assembly's `include::` list or heading structure
changed.

Never hand-edit `topic-registry.yaml`. An edit there is either something that belongs in a
fragment attribute — in which case put it there and regenerate — or something that belongs in
`topic-registry-meta.yaml`. A hand edit to the registry itself is erased by the next
regeneration, and until then it makes the registry disagree with the corpus it describes.

### Who Regenerates

- **Claude (chat or Claude Code)**: Edits fragments, then regenerates the registry per this
  standard as the last step of any topic work.
- **Authors**: Review the fragment diff. The registry diff is a consequence of it, and should
  contain nothing that the fragment diff does not explain.

### Validation

Generation removes a whole class of check. A registry produced from the corpus cannot hold a
`file` that does not resolve, a `topic-count` that disagrees with its entries, a duplicate
`topic-id`, or an entry for a file that is not there — none of those states is reachable. What
remains are the checks over *authored* values, which generation carries through faithfully and
therefore cannot correct:

1. All `dependencies` reference valid `topic-id` values. A dangling one means a fragment
   contains an `xref:` to a topic that does not exist — a broken link in the corpus, surfaced
   here.
2. All `related` entries reference valid `topic-id` values, and the relationship is
   reciprocated: if topic A lists topic B as `related`, topic B must list topic A.
3. Every topic with a non-empty `related` also carries a `scope-note`.
4. All `topic-refs` in assembly sections reference valid `topic-id` values. Since sections are
   read from the assembly document, a violation means the assembly `include::`s a file whose
   anchor names no known topic.
5. Every published topic outside the `ext` domain appears in at least one assembly's
   `sections`.
6. No `ext` topic appears in any assembly's `sections`, and no `ext` topic has
   `status: published`.

Rules 5 and 6 replace the former single rule "every published topic appears in at least one
assembly." As written, that rule contradicted the assembly exclusion rule in `IKE-INGEST.md`:
external topics must never appear in an assembly, so a published `ext-*` topic could satisfy
neither requirement. External topics are capped at `status: review`, so a published one is
itself the defect — rule 6 reports it as that rather than reporting a phantom missing
assembly.

The regeneration itself is the strongest check available: regenerate into a temporary location
and compare against the committed registry. Any difference is either a stale registry or a
fragment edited without regenerating, and in both cases the generated output is correct.

## Generated Artifact: term-index.yaml

The build produces `term-index.yaml` by collecting all `indexterm` and `((...))` entries from
topic `.adoc` files. This file is a generated artifact — it must not be hand-edited. See
`IKE-INDEX.md` for the full schema and authoring conventions.

### Purpose

The term index provides a reverse mapping from technical terms to topics. While the registry's
`keywords` and `summary` fields capture what a topic is *about*, the term index captures what
a topic *discusses*. This distinction matters for redundancy detection: two topics may have
different keywords but discuss the same underlying concepts.

### Location

```
{topic-library-module}/target/generated/term-index.yaml
```

The term index is generated during the build and placed in the `target/` directory. It is not
committed to source control. It is included in the packaged topic library zip so that
dependent modules and Claude have access to it after unpacking.

### Build Integration

A build-time script (Groovy, Python, or similar) walks all `.adoc` files under `topics/`,
extracts `indexterm` macros and `((...))` inline index terms, and produces the YAML file.
This script should run during the `process-resources` phase, after topic files are in place
but before packaging.

## Working with Claude

### Providing Context

At the start of a session involving topic work, upload or paste:

1. The `topic-registry.yaml` file (or the relevant domain section if the full file is too
   large).
2. The `term-index.yaml` file, if available and if the session involves integration or
   redundancy checking.

For a 600-page compendium decomposed into ~300 topics, the registry will be roughly 20–30 KB
of YAML and the term index roughly 10–15 KB — both well within context window limits.

Together, these two files give Claude a complete map of what exists (registry), where it sits
structurally (assembly sections), and what specific terms each topic discusses (term index).

### Requesting Topic Lookup

To find existing content without uploading topic files:

> Which topics cover STAMP coordinates? (Check the registry.)

Claude will search the registry's `title`, `keywords`, and `summary` fields to identify
matching topics and report their `topic-id`, `title`, and `summary`.

### Requesting Registry Updates

After any topic creation or modification:

> Regenerate the registry.

Claude rebuilds `topic-registry.yaml` from the fragments per § "The registry is derived, not
authored," merging in `topic-registry-meta.yaml`. Do not ask for a YAML fragment to paste into
the registry by hand — that reintroduces the second authored copy this design removes.
