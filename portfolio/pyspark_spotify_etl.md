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

<h4>Pipeline Overview</h4>

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

<h4>Step 1 — Spotify API: Extracting the Data</h4>

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

<h4>Step 2 — Landing Zone: Preserving Raw Data</h4>

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

<h4>Step 3 — Bronze Layer: Raw Ingestion into Delta Tables</h4>

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

<h4>Step 4 — Silver Layer: Cleaning and Star Schema Modeling</h4>

<p align='justify'>
The Silver layer is where the real data modeling happens. The nested Bronze data is flattened, cleaned, and restructured into a proper <strong>star schema</strong> — a standard pattern in data warehousing where one central fact table is surrounded by dimension tables.
</p>

**Why a Star Schema?**

The raw Spotify data has many-to-many relationships: one album can have multiple artists, and one artist can appear on multiple albums. The same applies to images. If you tried to store this in a single flat table, you would either:
- **Duplicate album rows** (one row per artist per album) — inflating the table and making aggregations wrong
- **Concatenate arrays as strings** — losing the ability to filter or join properly

A star schema solves this cleanly with **bridge tables** that sit between the fact and dimension tables.

**The Schema**

| Table | Type | Description |
|---|---|---|
| `fact_albums` | Fact | One row per album — release date, track count, album type |
| `dim_artists` | Dimension | One row per unique artist — name, URI, external links |
| `dim_images` | Dimension | One row per image — URL, width, height |
| `bridge_artists_albums` | Bridge | Resolves many-to-many between artists and albums |
| `bridge_images_albums` | Bridge | Resolves many-to-many between images and albums |

**Date Cleaning**

<p align='justify'>
Spotify sometimes returns partial release dates — for example <code>"2023"</code> (year only) or <code>"2023-04"</code> (year and month). The Silver layer handles this explicitly by detecting the date format length and parsing accordingly, then deriving a <code>release_year</code> column for use in the Gold layer.
</p>

**Data Quality Checks**

Before promoting data from Silver to Gold, the pipeline runs two types of checks:

- **Referential integrity** — every `artist_id` in `bridge_artists_albums` must exist in `dim_artists`. Any orphaned foreign key raises an error and halts the pipeline.
- **Null checks** — critical columns like `album_id`, `album_name`, and `release_date` are checked for nulls before they can corrupt Gold.

***

<h4>Step 5 — Silver Layer: Broadcast Joins</h4>

<p align='justify'>
When joining the bridge tables against fact and dimension tables, the pipeline uses <strong>broadcast joins</strong> for the dimension tables. Dimension tables like <code>dim_artists</code> are small — broadcasting them means Spark sends a copy to every executor, eliminating network shuffle entirely and making joins significantly faster.
</p>

***

<h4>Step 6 — Gold Layer: Aggregations and Growth Analysis</h4>

<p align='justify'>
The Gold layer is the analytics-facing output of the pipeline. It reads from the Silver star schema and produces two aggregated tables optimized for reporting and analysis.
</p>

**`gold_artists`** — artist-level rollup: total albums, average tracks per album, latest release date.

**`gold_artist_growth`** — year-by-year album count per artist with year-over-year growth percentage, calculated using PySpark window functions (`LAG()`).

**Z-ORDER Optimization** — after writing Gold tables, `OPTIMIZE ... ZORDER BY` physically co-locates records by `artist_id` and `release_year`, enabling Delta Lake to skip entire files on filtered queries.

***

<h4>Infrastructure</h4>

**Docker** — the entire pipeline runs inside a container, eliminating environment differences between machines.

**GitHub Actions** — triggers on every push to main, runs all four stages in sequence. If any stage fails, the run is marked failed immediately.

***

<h4>Key Engineering Decisions</h4>

| Decision | Why |
|---|---|
| Landing Zone before Bronze | Raw data always preserved — reprocess anytime without re-calling the API |
| Delta MERGE instead of overwrite | Incremental loads — no duplicates, pipeline stays fast as dataset grows |
| Star schema in Silver | Handles many-to-many relationships cleanly without row duplication |
| Bridge tables | Correct way to model artist ↔ album and image ↔ album relationships |
| Data quality checks between layers | Catches referential integrity and null issues before they corrupt Gold |
| Broadcast joins for dimensions | Eliminates shuffle for small tables — significant speedup |
| Z-ORDER on Gold tables | Data skipping reduces I/O on filtered queries at scale |

***

<h4>Tech Stack</h4>

**Python · PySpark · Delta Lake · Spotify Web API · Docker · GitHub Actions**

***

<p align='justify'>
<a href="https://github.com/MLArchitect/pyspark-spotify-etl" target="_blank">View the code on GitHub →</a>
</p>
