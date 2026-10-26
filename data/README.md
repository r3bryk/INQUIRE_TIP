# data/ – working folder

This is the working folder of the pipeline (`DATA_DIR = Path("../data")` in all three notebooks).

Put the input files here (see `docs/input_data.md`), or copy the content of `example_data/` here to test the pipeline:

```
data/
├── tox_vs_dust_merged.csv              merged tox vs. dust dataset (Parts 1 and 3)
├── feature_metadata_GCxGC-MS.xlsx      feature metadata, GCxGC-MS features (Part 1)
├── feature_metadata_LC-MS.xlsx         feature metadata, LC-MS features (Part 1)
├── feature_metadata_classified.xlsx    feature metadata of all features + chemical class (Parts 1 and 3)
└── inquire_tip_outputs/                created by the notebooks
    ├── <yymmdd>_*.csv / .xlsx          Part 1 outputs
    ├── endpoint_models/<endpoint>/     Part 2 outputs per endpoint
    ├── cross_endpoint_summary/         Part 2 cross-endpoint outputs
    └── inquire_tip_figures/            Part 3 figures
```

Everything in this folder except this README is excluded from the repository (`.gitignore`), so real data and results are never uploaded by accident.
