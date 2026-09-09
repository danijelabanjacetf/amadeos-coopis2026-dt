# From Business Process Models to Runnable Applications by AI-empowered AMADEOS

Companion artifact for the **CoopIS-DT 2026** tool-demonstration paper. This repository
documents the *Generate app* feature of the AMADEOS tool, which turns a conceptual data
model (CDM) — automatically derived from business process models — into a
[JHipster](https://www.jhipster.tech) Domain Language (JDL) specification, from which a
full-stack web application can be generated (the user selects the framework, frontend,
authentication, and database options interactively in JHipster).

**Authors:** Danijela Banjac, Zoran Djuric, Drazen Brdjanin, Mihajlo Savic, Goran Banjac
**Affiliation:** Faculty of Electrical Engineering, University of Banja Luka, Bosnia and Herzegovina
**Venue:** International Conference on Cooperative Information Systems (CoopIS) 2026 — Tool Demonstration Track

## Live demo

- **Platform:** https://mlabplatform.online/
- **Demo credentials:** `coopis2026` / `coopis2026`
- **Demonstration video:** https://drive.google.com/drive/folders/1RR_33hExMJ8AdkTCjc4yc7dt2yS7R9oa?usp=sharing

Authenticated users can inspect and download the artifacts of the example projects without
preparing their own source BPMs.

---

## What this repository documents

The *Generate app* transformation proceeds in two stages:

1. a **deterministic** rule-based transformation (always run), and
2. an **optional** LLM-based enrichment pass (disabled by default).

The deterministic stage produces the guaranteed baseline for supported, well-formed CDMs.
The enrichment stage only augments that baseline; if it fails or returns empty output, the
deterministic JDL is returned unchanged. The CDM is received as the
[mxGraph](https://github.com/jgraph/mxgraph) cell model (`<mxGraphModel>` / `<mxCell>`)
exported from the embedded diagram editor.

---

## 1. Deterministic mapping rules

The mapper emits `entity` and `relationship` blocks only — **no `application {}` block** is
generated, so the user chooses the framework, authentication, and database options
interactively when running `jhipster jdl`.

### 1.1 Entities and attributes

- Each CDM class becomes a JDL `entity` block.
- Each attribute becomes a field `<name> <type>`, with the type mapped per the table below.
- **Dropped attributes:**
  - An attribute named `id` (case-insensitive) is **dropped** — JHipster generates its own
    primary key and rejects a JDL field literally named `id`.
  - An attribute whose type matches another class's name is **not** emitted as a field; it
    is a reference and is carried by a relationship instead.

### 1.2 Attribute type mapping

The CDM type is matched case-insensitively; unrecognized or empty types default to
`String`.

| CDM type (case-insensitive)   | JDL type    |
| ----------------------------- | ----------- |
| `Integer`                     | `Integer`   |
| `Text`, `String`              | `String`    |
| `Double`, `Real`, `Float`     | `Double`    |
| `Date`                        | `LocalDate` |
| `Logical`, `Boolean`          | `Boolean`   |
| `Blob`                        | `Blob`      |
| *(empty / unrecognized)*      | `String`    |

### 1.3 Identifier sanitization

- Class names → **PascalCase**; field and relationship reference names → **camelCase**.
- Names are tokenized on non-alphanumeric characters; tokens are capitalized and joined.
- Identifiers are forced to start with a letter (a JDL identifier cannot lead with a
  digit); if nothing usable remains, a fallback (`Entity` / `field`) is used.
- Name collisions are resolved by appending a numeric suffix (`Name`, `Name2`, `Name3`, …).

### 1.4 Relationships and cardinality

Each CDM association becomes a JDL `relationship` block. An association **end** is
classified as:

- **to-one** if its multiplicity is `1` or `0..1` (no `*`), and
- **to-many** otherwise (multiplicity contains `*`).

If an end has **no** multiplicity label, a default is assumed so that a fully unlabeled
association becomes the common `ManyToOne`: a missing **source** end defaults to *to-many*,
a missing **target** end defaults to *to-one*.

The relationship type follows from the two ends:

| Source end | Target end | JDL relationship |
| ---------- | ---------- | ---------------- |
| to-one     | to-many    | `OneToMany`      |
| to-many    | to-one     | `ManyToOne`      |
| to-one     | to-one     | `OneToOne`       |
| to-many    | to-many    | `ManyToMany`     |

- Reference field names default to the camelCased name of the other class.
- A `required` constraint is added on an end whose multiplicity is **exactly `1`**
  (i.e. mandatory, not `0..1`). `ManyToMany` carries no `required` (JHipster does not
  support it on either owning side).

### 1.5 Well-formedness and failure modes

The deterministic stage produces a valid baseline **for supported, well-formed CDMs** —
classes with valid identifiers and attributes whose types are covered above. It is not an
unconditional guarantee: malformed or unsupported inputs yield correspondingly limited
output (unrecognized types fall back to `String`, the reserved `id` is dropped,
entity-typed attributes are reclassified as relationships). Flawed input models produce
flawed output models.

---

## 2. JHipster version

Generated specifications were produced and built with **JHipster 9.2.0**
(`generator-jhipster@9.2.0`).

The generated file contains only `entity` and `relationship` (and, after enrichment,
`enum` and option) declarations — no `application {}` block. It is applied to a scaffolded
JHipster application, e.g.:

```bash
# 1. scaffold and configure the application (creates .yo-rc.json)
jhipster

# 2. import the generated entities into that application
jhipster jdl app.jdl
```

Alternatively, in a fresh folder, a base name must be supplied because the file has no
`application {}` block:

```bash
jhipster jdl app.jdl --defaults --base-name myApp
```

---

## 3. LLM configuration

| Setting            | Value                                                             |
| ------------------ | ----------------------------------------------------------------- |
| Enrichment         | Optional, **disabled by default**                                 |
| Gateway            | [OpenRouter](https://openrouter.ai) (OpenAI-compatible)           |
| Model selection    | Provider-independent, selectable **per request**                  |
| Default model      | `deepseek/deepseek-v4-flash`                                      |
| Temperature        | `0.2`                                                             |
| Request shape      | Two-message chat: `system` + `user`                               |
| Model inputs       | Source CDM (mxGraph XML) **and** deterministic JDL                |
| Post-processing    | Strip accidental code fences; normalize `pagination` → `paginate` |
| Failure handling   | Empty/failed response ⇒ fall back to deterministic JDL            |
| Credential         | Server-side only (`OPENROUTER_API_KEY`); never sent to the client |

---

## 4. Representative prompt

The enrichment request is a two-message chat. The `%s` placeholders are substituted with
the source CDM (mxGraph XML) and the deterministic JDL, respectively.

### System message

```
You are a JHipster JDL expert. You are given a valid JHipster JDL derived from a data
model. Improve it without breaking it. Output ONLY valid JDL — no explanations, no
markdown, no code fences.
```

### User message

```
You are given two representations of the same domain:
(1) the source conceptual data model (CDM), serialized as mxGraph XML, which is the
authoritative description of the domain — its entities, any modeled attributes, and the
relationships between them; and
(2) a JHipster JDL that was mechanically derived from that CDM, which is the structural
baseline you must preserve and extend.

Enhance the JDL by:
- for any entity that has few or no attributes, proposing a set of plausible attributes
  inferred from the CDM and from the entity's name and its relationships, using valid JDL
  types; do NOT add attributes to entities that already declare a meaningful set of their
  own, and never add an attribute that merely duplicates an existing relationship
  reference;
- adding field validations where clearly appropriate: required for non-optional fields,
  minlength/maxlength for strings, min/max for numbers;
- converting fields that are obviously categorical (e.g. status, type, state, role,
  category) into JDL enum types, and defining each enum with plausible values;
- appending exactly these three option lines at the very end of the file, using this
  literal syntax, in this exact order, with "all" as the entity list:
  service all with serviceImpl
  dto all with mapstruct
  paginate all with infinite-scroll
Use the literal keyword "paginate" exactly as shown (this is the only pagination keyword
the JDL parser accepts — do not write "pagination" instead, it will fail to parse). Keep
"service" before "dto" (in that order).

Rules:
- Preserve EVERY existing entity, field, and relationship (same names). You MAY add new
  attributes to sparse entities as described above, but never remove or rename an existing
  field, and keep existing field types unless an enum conversion is clearly warranted. Do
  NOT change a field's type for any other reason (e.g. do not "upgrade" a Double to a more
  precise type).
- The ONLY valid JDL field types are: String, Integer, Long, BigDecimal, Float, Double,
  UUID, Boolean, LocalDate, ZonedDateTime, Instant, Duration, LocalTime, Blob, AnyBlob,
  ImageBlob, TextBlob, or an enum type you defined. There is no "Decimal" type — the
  precise decimal type is called "BigDecimal". Never invent a type name outside this list.
- Do NOT invent new entities or relationships, and do NOT add an application {} block.
  Ignore any diagram layout, styling, or geometry in the CDM — only its semantics matter.
- Output ONLY the JDL text — no prose, no code fences.

CDM (mxGraph XML):
%s

JDL:
%s
```

---

## Citation

> D. Banjac, Z. Djuric, D. Brdjanin, M. Savic, and G. Banjac, "From Business Process Models to Runnable Applications by AI-empowered AMADEOS," in *Proc. Int. Conf. on Cooperative Information Systems (CoopIS), Tool Demonstration Track*, 2026.

```bibtex
@inproceedings{Banjac2026Amadeos,
  title     = {From Business Process Models to Runnable Applications by AI-empowered AMADEOS},
  author    = {Banjac, Danijela and Djuric, Zoran and Brdjanin, Drazen and Savic, Mihajlo and Banjac, Goran},
  booktitle = {Proceedings of the International Conference on Cooperative Information Systems (CoopIS), Tool Demonstration Track},
  year      = {2026}
  % publisher, pages, and doi to be added once the proceedings are published
}
```