# eodh-qa-data

QA documentation and radiometric uncertainty data files for EODH collections, produced by NPL.

---

## What this repo contains

NPL produces two types of quality assessment for data collections in the EODH catalogue:

| Type | STAC asset key | Description | Frequency |
|------|----------------|-------------|-----------|
| Quality Processes Review (QPR) | `qa_documentation` | Assesses methodology, documentation, and processes used to produce the data | One per collection |
| Radiometric uncertainty | `qa_radiometric` | Measures sensor calibration and radiometric accuracy over time | One consolidated file per collection |

Both are JSON files produced by NPL workflows running in the NPL workspace.

## Collections covered

| Collection | EODH catalogue ID | QPR | Radiometric uncertainty |
|------------|-------------------|-----|------------------------|
| Sentinel-2 L1C | `sentinel2_l1c` | yes | yes |
| Airbus Pleiades HR | `airbus_phr_data` | yes | yes |
| Airbus Pleiades Neo | `airbus_pneo_data` | yes | no |
| Airbus SPOT | `airbus_spot_data` | yes | no |
| PlanetScope Scenes | `PSScene` | yes | yes |
| SkySat Collect | `SkySatCollect` | yes | no |

> **Note on Sentinel-2:** NPL QA records assess Sentinel-2 L1C data, but the EODH catalogue currently contains Sentinel-2 ARD rather than L1C.

---

## Repository structure

```
eodh-qa-data/
  qa-collection-map.json
  qa_documentation/
    {qa_key}_qa_check_quality_processes_review.json
  qa_radiometric/
    {qa_key}_qa_check_radiometric_unc_all_dates.json
```

### File naming

| Asset type | File name pattern |
|------------|------------------|
| QPR | `{qa_key}_qa_check_quality_processes_review.json` |
| Radiometric uncertainty | `{qa_key}_qa_check_radiometric_unc_all_dates.json` |

Examples for a collection with `qa_key` of `airbus_phr_data`:

- `qa_documentation/airbus_phr_data_qa_check_quality_processes_review.json`
- `qa_radiometric/airbus_phr_data_qa_check_radiometric_unc_all_dates.json`

> **Note:** Earlier iterations produced per-year radiometric files (e.g. `..._2022-12-01_2022-12-31.json`). These have been consolidated into single per-collection files. The transformer expects the consolidated single-file format only.

### File content

The transformer does not validate or parse the contents of QA JSON files — it only checks that they exist. No specific schema is enforced at ingestion time.

---

## The collection map

`qa-collection-map.json` maps STAC collection IDs to QA keys. This is needed when the QA file prefix differs from the STAC collection `id`:

```json
{
  "sentinel2_ard": "sentinel2_l1c"
}
```

If a collection ID has no entry in the map, the collection `id` is used as the QA key directly.

**To add a new collection:** add an entry to `qa-collection-map.json` and deposit the corresponding QA files in the correct directories. No front-end changes are needed.

---

## How harvest-transformer uses this repo

The `harvest-transformer` service attaches QA assets to STAC collection records. When a collection is processed it:

1. Takes the collection's STAC `id` field.
2. Looks up the `id` in `qa-collection-map.json` to find the QA key (falls back to the `id` itself if not found).
3. Constructs expected raw GitHub URLs for both QA file types and checks whether each file exists (HTTP HEAD request).
4. Adds a `qa_documentation` and/or `qa_radiometric` asset entry for each file that exists.

Assets are only ever added, never overwritten.

The transformer is configured via two environment variables pointing at this repo:

| Variable | Example value |
|----------|---------------|
| `QA_COLLECTION_MAP_URL` | `https://raw.githubusercontent.com/EO-DataHub/eodh-qa-data/main/qa-collection-map.json` |
| `QA_ASSET_ROOT` | `https://raw.githubusercontent.com/EO-DataHub/eodh-qa-data/main` |

---

## When will an asset appear in the catalogue?

QA assets are attached to a collection the next time that collection is harvested. Depositing a file here does not immediately update the catalogue — the change takes effect on the next scheduled harvest run.

| Harvester | Collections | Frequency | Prod (UTC) | Staging (UTC) | Test (UTC) |
|-----------|-------------|-----------|------------|---------------|------------|
| CEDA (config harvester) | CEDA catalogue | Daily | 06:00 | 06:00 | 06:00 |
| Planet | `PSScene`, `SkySatCollect` | Daily | 07:30 | 07:30 | 07:30 |
| Airbus SAR | `airbus_sar_data` | Monthly | 20th 02:30 | 16th 13:20 | 13th 15:00 |
| Airbus SPOT | `airbus_spot_data` | Monthly | 20th 03:00 | 16th 13:20 | 13th 15:00 |
| Airbus PNEO | `airbus_pneo_data` | Monthly | 20th 03:30 | 16th 13:20 | 13th 15:00 |
| Airbus PHR | `airbus_phr_data` | Monthly | 20th 04:00 | 16th 13:20 | 13th 15:00 |

Airbus collections are harvested monthly — a file deposited shortly after a harvest run could wait up to a month before appearing in the catalogue.

### Forcing a re-harvest

Each harvester tracks what it has already processed by storing a hash of each file in an S3 folder called `harvested-metadata`. On every run it compares the current hash against the stored one and only re-processes files that have changed.

> **Automatic cache invalidation:** merging a change to `qa_documentation/` or `qa_radiometric/` into `main` automatically invalidates the `harvested-metadata` entries for the affected collections, so the next scheduled harvest picks up the change without manual intervention.

---

## Resulting STAC asset structure

Once the transformer runs, the collection record gains assets such as:

```json
"assets": {
  "qa_documentation": {
    "href": "https://raw.githubusercontent.com/EO-DataHub/eodh-qa-data/main/qa_documentation/airbus_phr_data_qa_check_quality_processes_review.json",
    "type": "application/json",
    "title": "Quality Processes Review",
    "roles": ["metadata", "quality"]
  },
  "qa_radiometric": {
    "href": "https://raw.githubusercontent.com/EO-DataHub/eodh-qa-data/main/qa_radiometric/airbus_phr_data_qa_check_radiometric_unc_all_dates.json",
    "type": "application/json",
    "title": "Radiometric Uncertainty Assessment",
    "roles": ["metadata", "quality"]
  }
}
```
