# Vessel Identification & AI Agent - Design Notes

Design notes for resolving vessel identities from the dataset and for an
AI-powered search layer on top. Per the brief this is design + small code, not
a full build. The runnable bits are in the two notebooks:

- `notebooks/01_data_exploration.ipynb` - what the columns mean, how dirty the data is
- `notebooks/02_identity_resolution.ipynb` - validate / merge / fuzzy-match / SQL search

Where a real component is implied I name the tool and move on rather than build it.

## The data

1,734 rows, 50 columns. The columns are really four kinds of thing mixed into
one wide row, and the matching strategy depends on keeping them apart:

- identity: imo, mmsi, name, callsign, flag, vessel_type
- static / registry: dimensions, tonnage, build, engine (changes rarely)
- dynamic / AIS: position, speed, destination, eta (a sighting, not identity)
- bookkeeping: InsertDate, UpdateDate

What the exploration found (numbers in notebook 1):

- IMO fails the checksum on 670 / 1734 rows. But ~617 of those are placeholders
  (0, 1000000, 123456789 reused across many rows) and only ~53 are real typos.
  "IMO unknown" and "IMO mistyped" are different problems.
- MMSI is unique on every row but unstable: 196 valid IMOs map to more than one
  MMSI (one ship has 28).
- Duplication is entirely MMSI change. Zero duplicate (imo, mmsi) pairs, and all
  196 duplicated valid IMOs are duplicated because the MMSI changed. So the file
  is current-state snapshots, not a position time-series.
- draught appears twice under one name - design draught (static) and AIS-reported
  draught. 614 of 633 overlapping rows differ, so they're different fields.
- Invalid-IMO rows are still real ships (~90% have name/dims/flag). Dropping them
  loses 39% of the data, so they get a fallback, not deletion.

## Identity resolution

The whole thing turns on one question: is the IMO trustworthy? That routes a row
to a deterministic path or a fuzzy one.

```
record -> validate IMO
            valid     -> merge by IMO -> golden record + history
            invalid   -> fuzzy match on static fields -> link or new vessel
```

### Validate (Key Q2)

IMO checksum: 7 digits, 7th is the check digit. Cheap, catches typos. Then split
invalid into sentinel vs dirty data-drivenly - an invalid value reused on many
rows is a placeholder ("no key"); a one-off invalid value is a recoverable typo.

### Merge valid IMOs (Key Q1, Q3)

Since duplication among valid IMOs is only MMSI change, "same valid IMO = same
ship" holds with no guessing. Collapse a ship's rows into one golden record plus
a change history. The rule depends on the field:

- immutable (length, tonnage, build year): a hull doesn't change, so disagreement
  is a data error. Fill gaps from any row, pick by majority (tie -> latest), and
  flag the conflict. The losing value is wrong, not historical - drop it.
- mutable (name, flag, mmsi, callsign): these really change. Keep the latest as
  current, but store the past values as history. Otherwise a lookup for an old
  MMSI or former flag finds nothing - which is the answer to "track changes".

In the data: 1,064 valid rows -> 668 vessels, 196 with a real MMSI history, 78
with a flagged immutable conflict.

### Fuzzy match keyless rows (Key Q1)

No key, but usually a name + dimensions. Score against the golden records on
static fields only (name 0.45, length 0.20, width 0.15, gross tonnage 0.10,
callsign 0.10). Position is never used - being in the same place is coincidence,
not identity.

Three choices worth calling out:

- block first (compare within the same vessel_type) to keep it cheap. Real
  version: length-bucket + name-prefix, or an ANN index.
- conservative threshold. Below it, the row becomes a new vessel instead of being
  force-merged. A wrong merge corrupts two ships permanently; a missed one can be
  retried. False merges are the worse error.
- return the evidence with the score (e.g. name~1.00, length=, GT=) so a decision
  is auditable - and reusable as grounding for the AI layer.

Examples from the notebook: a placeholder-IMO row named FREEDOM links to a real
vessel at 0.75; a 2-char name "XF" scores 0.11 and stays a new vessel.

### Reverse conflict: one MMSI, several IMOs

Rare here. Handled the same way - a MMSI under two valid IMOs is flagged for
review, never auto-merged. Two real hulls can reuse a recycled MMSI at different
times.

## Ground truth (Key Q4)

A fully-automated perfect ground truth isn't realistic from this data alone:
the dirty rows have nothing to check against. What's realistic is a canonical
store with provenance and confidence:

- valid-IMO golden records are high-confidence, deterministic
- fuzzy links carry their score + evidence, so they read as "probable", not "true"
- cross-check against an external registry (IHS / Equasis / public IMO) where
  licensing allows
- low-confidence and conflicting cases go to a human review queue

So not one oracle, but a confidence-scored, source-attributed, correctable record.

## System design (Key Q5)

```
 sources(static+AIS) -> ingestion -> identity resolution -> canonical store
                                                              |
                                            +-----------------+-----------------+
                                            |                                   |
                                       SQL filters                        text/vector search
                                            |                                   |
                                            +-----------------+-----------------+
                                                              |
                                                     conversational AI (LLM + tools)
```

Storage: PostgreSQL. The data is structured, queries are filter-heavy, and the
IMO/MMSI/history relationships are relational. PostGIS for position and port
proximity. Three tables, because mixing current state and history is what makes
this data unmanageable:

```sql
CREATE TABLE vessels (
    imo            INTEGER PRIMARY KEY,   -- validated IMO; NULL for keyless ships
    vessel_uid     UUID NOT NULL,         -- internal id for ships with no IMO
    name           TEXT,
    flag           TEXT,
    vessel_type    TEXT,
    mmsi           BIGINT,                -- current MMSI
    length         REAL,
    gross_tonnage  REAL,
    match_confidence REAL,               -- 1.0 deterministic; <1.0 fuzzy-linked
    updated_at     TIMESTAMPTZ
);

CREATE TABLE vessel_history (            -- append-only changes to mutable fields
    vessel_uid  UUID REFERENCES vessels(vessel_uid),
    attribute   TEXT,                    -- 'mmsi' | 'flag' | 'name'
    old_value   TEXT,
    new_value   TEXT,
    observed_at TIMESTAMPTZ
);

CREATE TABLE positions (                 -- AIS, never used for identity
    vessel_uid  UUID REFERENCES vessels(vessel_uid),
    latitude    REAL, longitude REAL, speed REAL,
    observed_at TIMESTAMPTZ
);
```

A filter query - the thing the AI layer ends up emitting (see notebook 2):

```sql
SELECT imo, name, flag, length, gross_tonnage
FROM vessels
WHERE vessel_type = 'Crude Tanker' AND length > 250 AND flag = 'LR'
ORDER BY length DESC;
```

A former-identifier lookup, which is why history is its own table:

```sql
-- ship that used to broadcast MMSI 538007248
SELECT v.* FROM vessels v
JOIN vessel_history h USING (vessel_uid)
WHERE h.attribute = 'mmsi' AND h.old_value = '538007248';
```

Search has two paths because users ask in two ways: structured filters go to SQL;
fuzzy name lookup ("the ship called Sanmar something") goes to a text index
(pg_trgm is enough at this scale; Elasticsearch later if needed). Name embeddings
in pgvector cover typo-tolerant search and double as the matching index above.

On tooling: start with one PostgreSQL + extensions, not a search cluster - a few
thousand vessels don't need it, and one system is easier to keep consistent. Add
Elasticsearch / a vector DB when scale forces it. Python + pandas for the batch
resolution job; rapidfuzz instead of difflib in production (faster; difflib is
just for the demo).

## Conversational AI (Key Q6, Q7)

The LLM doesn't answer from memory. It turns a question into a tool call, the DB
answers, and the LLM formats the result.

```
"bulk carriers near Singapore over 200m"
  -> search_vessels(type="Dry Bulk", min_length=200, near="Singapore")
  -> parameterized SQL against the store
  -> rows -> LLM formats, citing the IMO/MMSI from the rows
```

This is also how hallucination is prevented (Key Q7):

- the model only emits structured tool calls, not free-form facts - the DB
  produces the data, not the model
- answers are grounded: the LLM may only state what the rows contain, and cites
  the IMO/MMSI used. Empty result -> "no matching vessels", not an invented ship.
- confidence is shown: a fuzzy link is presented as "probable match (0.75, on
  name + dimensions)", reusing the evidence from the matcher
- enums (vessel_type, flag) and ranges are validated before the query runs

Caching and session (from the brief): cache repeated lookups keyed on the
normalized query, invalidate when the record's updated_at changes. Session state
holds the resolved entities in the conversation ("that ship" -> the IMO two turns
back) and the last result set, so follow-up filters compose without re-querying.

## Evaluation (Key Q8)

Identity resolution - needs a labelled sample (a few hundred adjudicated pairs):

- precision / recall of the fuzzy matcher (precision matters more - false merges
  are the costly error)
- the valid/sentinel/dirty split checked against a known registry
- count of unresolved conflicts left for review

AI search - judged on retrieval, not prose:

- intent -> query accuracy: does the question map to the right filters? (fixed
  question -> expected-query test set)
- groundedness: every fact in the answer traces to a returned row (automatable;
  LLM-as-judge at scale)
- retrieval precision / recall against expected result sets
- refusal correctness: empty result -> "none found", never a fabricated vessel

## Trade-offs, short version

- IMO is the primary key, but validate before trusting it
- MMSI is an attribute with history, never a key
- deterministic where the data allows, fuzzy only where there's no key (~39%)
- merge conservatively - a missed link is recoverable, a false merge isn't
- one database first, specialized stores when scale demands
- the LLM routes and narrates; it never owns the facts
