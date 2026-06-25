---
layout: page
title: ""
subtitle: "How do you track which artists are growing? You build a pipeline."
cover-img: 
thumbnail-img: 
share-img: 
tags: [PySpark, Delta Lake, Spotify API, ETL, Data Engineering, Docker, GitHub Actions]
---

<h4>PySpark Spotify ETL Pipeline</h4>

<p align='justify'>
I built this project to prove a point: even a fun dataset like Spotify releases deserves production-grade engineering. The result is a pipeline that automatically tracks new album releases, models artist activity over time, and surfaces year-over-year growth trends — giving a music label, streaming platform, or analyst a reliable feed of market intelligence without touching the API more than once.
</p>

***

## Pipeline Overview

<p align="center">
<img src="/assets/portfolio/spotify-etl-architecture.png" width="900" alt="Medallion Architecture Diagram">
</p>

The pipeline follows a **medallion architecture** with four stages:

| Stage | What Happens |
|---|---|
| **Landing Zone** | Raw JSON responses from the Spotify API are saved to disk, unchanged |
| **Bronze** | JSON files are parsed and loaded into Delta tables — minimal transformation |
| **Silver** | Data is cleaned, flattened, and modeled into a star schema |
| **Gold** | Silver tables are aggregated into artist-level metrics and growth analysis |

Each stage is independent and can be re-run without affecting the others. This is a deliberate design choice — if a bug is found in the Silver transformation, you can fix and reprocess without re-calling the API or touching Bronze.

***

## Step 1 — Spotify API: Extracting the Data

<p align='justify'>
The Spotify Web API provides a <code>/browse/new-releases</code> endpoint that returns a paginated list of newly released albums. Authentication uses the <strong>Client Credentials flow</strong> — the pipeline requests a temporary access token using a client ID and secret, then uses that token to call the API.
</p>

The API response is a **nested JSON structure** — each album contains an array of artists, an array of cover images at different resolutions, and various metadata fields. This nesting is what makes the data non-trivial to work with: a single album can have multiple artists and multiple images, so a naive flat table would either duplicate rows or lose information.

A typical response for one album looks like:

```json
{
  "id": "abc123",
  "name": "Album Title",
  "release_date": "2024-03-15",
  "total_tracks": 12,
  "album_type": "album",
  "artists": [
    { "id": "artist1", "name": "Artist One" },
    { "id": "artist2", "name": "Artist Two" }
  ],
  "images": [
    { "url": "https://...", "width": 640, "height": 640 },
    { "url": "https://...", "width": 300, "height": 300 }
  ]
}
```

The pipeline paginates through all available pages and saves each page as a timestamped JSON file. **Nothing is transformed at this point** — the raw API response is saved exactly as received.

***

## Step 2 — Landing Zone: Preserving Raw Data

<p align='justify'>
Before any processing begins, every API response is written to a local <strong>Landing Zone</strong> directory as a timestamped JSON file. This is one of the most important design decisions in the pipeline.
</p>

Why? Because if a transformation bug is discovered later — say, a date parsing error in the Silver layer — you can fix the logic and reprocess from the Landing Zone without having to call the Spotify API again. The raw data is always preserved.

This also provides a full audit trail: you can see exactly what data arrived, when it arrived, and from which API call.

```
landing/
  new_releases_page0_20240315_120001.json
  new_releases_page1_20240315_120003.json
  new_releases_page2_20240315_120005.json
  ...
```

***

## Step 3 — Bronze Layer: Raw Ingestion into Delta Tables

<p align='justify'>
The Bronze layer reads all JSON files from the Landing Zone using PySpark and loads them into <strong>Delta Lake tables</strong>. The transformation here is minimal — the nested arrays are kept as-is, and two audit columns are added: <code>ingested_at</code> (timestamp of when the record was loaded) and <code>source_file</code> (which JSON file it came from).
</p>

The critical technique here is **Delta Lake MERGE** (also called an upsert). Instead of truncating and reloading the entire table on every run, MERGE compares incoming records against what already exists:

- If an album already exists → **update** it (in case metadata changed)
- If an album is new → **insert** it

This makes the pipeline **incremental**: each run only processes what is new or changed. The table grows over time without duplicating data, and the pipeline stays fast even as the dataset grows.

```
Bronze Delta Table: albums
├── album_id (key)
├── album_name
├── release_date
├── total_tracks
├── album_type
├── artists_raw   ← still nested array
├── images_raw    ← still nested array
├── ingested_at
└── source_file
```

***

## Step 4 — Silver Layer: Cleaning and Star Schema Modeling

<p align='justify'>
The Silver layer is where the real data modeling happens. The nested Bronze data is flattened, cleaned, and restructured into a proper <strong>star schema</strong> — a standard pattern in data warehousing where one central fact table is surrounded by dimension tables.
</p>

### Why a Star Schema?

The raw Spotify data has many-to-many relationships: one album can have multiple artists, and one artist can appear on multiple albums. The same applies to images. If you tried to store this in a single flat table, you would either:
- **Duplicate album rows** (one row per artist per album) — inflating the table and making aggregations wrong
- **Concatenate arrays as strings** — losing the ability to filter or join properly

A star schema solves this cleanly with **bridge tables** that sit between the fact and dimension tables.

### The Schema

<p align='justify'>The Silver layer produces five tables:</p>

| Table | Type | Description |
|---|---|---|
| `fact_albums` | Fact | One row per album — release date, track count, album type |
| `dim_artists` | Dimension | One row per unique artist — name, URI, external links |
| `dim_images` | Dimension | One row per image — URL, width, height |
| `bridge_artists_albums` | Bridge | Resolves many-to-many between artists and albums |
| `bridge_images_albums` | Bridge | Resolves many-to-many between images and albums |

### Date Cleaning

<p align='justify'>
Spotify sometimes returns partial release dates — for example <code>"2023"</code> (year only) or <code>"2023-04"</code> (year and month) for releases where the exact day is unknown. These partial strings will fail if parsed naively as a full date. The Silver layer handles this explicitly by detecting the date format length and parsing accordingly, then deriving a <code>release_year</code> column for use in the Gold layer.
</p>

### Data Quality Checks

Before promoting data from Silver to Gold, the pipeline runs two types of checks:

**Referential integrity** — every `artist_id` in `bridge_artists_albums` must exist in `dim_artists`. Any orphaned foreign key raises an error and halts the pipeline before bad data reaches Gold.

**Null checks** — critical columns like `album_id`, `album_name`, and `release_date` in `fact_albums` are checked for nulls. A null in a key column would break downstream joins silently, producing incorrect aggregations without any obvious error.

These checks run as code, not as database constraints, which means they catch problems early and produce clear error messages with counts of bad records.

***

## Step 5 — Silver Layer: Broadcast Joins

<p align='justify'>
When joining the bridge tables against fact and dimension tables, the pipeline uses <strong>broadcast joins</strong> for the dimension tables. In a distributed system like Spark, a standard join requires shuffling data across all executors so matching rows end up on the same machine — this is expensive for large tables.
</p>

Dimension tables like `dim_artists` are small (a few thousand rows at most). Broadcasting them means Spark sends a copy of the entire dimension table to every executor, so each executor can perform the join locally without any network shuffle. For small dimensions, this can be dramatically faster.

***

## Step 6 — Gold Layer: Aggregations and Growth Analysis

<p align='justify'>
The Gold layer is the analytics-facing output of the pipeline. It reads from the Silver star schema and produces two aggregated tables optimized for reporting and analysis.
</p>

### `gold_artists`
An artist-level rollup that answers questions like:
- How many albums has this artist released in total?
- What is their average number of tracks per album?
- When was their most recent release?

### `gold_artist_growth`
A year-by-year breakdown of how many albums each artist released per year, with a **year-over-year growth percentage** calculated using PySpark **window functions**.

Window functions let you compute values across a set of rows related to the current row — in this case, looking back at the previous year's album count for the same artist to calculate percentage growth. This is the same concept as SQL's `LAG()` function applied within a partition.

### Z-ORDER Optimization

After writing the Gold tables, the pipeline runs `OPTIMIZE ... ZORDER BY` on the key columns (`artist_id`, `release_year`). Z-ORDER is a Delta Lake feature that physically rearranges data in storage so records with similar values for those columns are stored together in the same files.

When a query filters on `artist_id`, Delta Lake can skip entire files that don't contain that artist — this is called **data skipping**. For large tables with many artists, this reduces I/O dramatically without any changes to the query or schema.

***

## Infrastructure

### Docker
<p align='justify'>
The entire pipeline — PySpark, Delta Lake, all Python dependencies — runs inside a Docker container. This eliminates environment differences between machines and ensures the same setup runs locally and in CI.
</p>

### GitHub Actions (CI/CD)
<p align='justify'>
A GitHub Actions workflow triggers on every push to the main branch. It builds the Docker image and runs all four pipeline stages in sequence: extract → bronze → silver → gold. If any stage fails, the workflow exits with an error and the run is marked as failed, making regressions immediately visible.
</p>

***

## Key Engineering Decisions

| Decision | Why |
|---|---|
| Landing Zone before Bronze | Raw data is always preserved — reprocess anytime without re-calling the API |
| Delta MERGE instead of overwrite | Incremental loads — no duplicates, pipeline stays fast as dataset grows |
| Star schema in Silver | Handles many-to-many relationships cleanly without row duplication |
| Bridge tables | Correct way to model artist ↔ album and image ↔ album relationships |
| Data quality checks between layers | Catches referential integrity and null issues before they corrupt Gold |
| Broadcast joins for dimensions | Eliminates shuffle for small tables — significant speedup |
| Z-ORDER on Gold tables | Data skipping reduces I/O on filtered queries at scale |

***

## Tech Stack

**Python · PySpark · Delta Lake · Spotify Web API · Docker · GitHub Actions**

***

<p align='justify'>
<a href="https://github.com/MLArchitect/pyspark-spotify-etl" target="_blank">View the code on GitHub →</a>
</p>
