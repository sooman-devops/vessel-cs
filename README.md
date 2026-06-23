# Vessel Identification & AI Agent - Case Study

Resolving vessel identities from the noisy registry/AIS dataset, plus a design
for an AI-powered search layer. Design + small code, not a full build.

## What's here

- `docs/design_document.md` - the main write-up. Identity resolution, system
  design, the conversational-AI layer, hallucination prevention, evaluation.
  Covers the eight Key Questions.
- `notebooks/01_data_exploration.ipynb` - what the columns mean and how dirty
  the identifiers are.
- `notebooks/02_identity_resolution.ipynb` - the matching logic: validate IMO,
  merge by IMO, fuzzy-match the keyless rows, search with SQL. Resolution is
  Python, search is SQL.

## Running

```
pip install -r requirements.txt
jupyter notebook    # open the files in notebooks/
```

The notebooks read `../data/...`, so run them from the `notebooks/` directory.

## Layout

```
.
├── README.md
├── requirements.txt
├── docs/design_document.md
├── notebooks/
│   ├── 01_data_exploration.ipynb
│   └── 02_identity_resolution.ipynb
└── data/case_study_dataset_202509152039.csv
```
