---
layout: page
title: PySpark Spotify ETL Pipeline
subtitle: End-to-end data pipeline with medallion architecture
cover-img: 
thumbnail-img: 
share-img: 
tags: [PySpark, Delta Lake, Spotify API, ETL, Data Engineering, Docker, GitHub Actions]
---

<p align='justify'>
I built an end-to-end data pipeline that pulls new album releases from the Spotify Web API and processes them through a <strong>medallion architecture</strong> using PySpark and Delta Lake.
</p>

***

## Architecture

<p align="center">
<img src="/assets/portfolio/spotify-etl-architecture.png" width="900" alt="Medallion Architecture Diagram">
</p>

The pipeline flows through three layers:

- **Bronze** — raw data ingested directly from the Spotify API and stored as Delta tables
- **Silver** — cleaned and modeled into a star schema (fact tables, dimension tables, and bridge tables for many-to-many relationships)
- **Gold** — aggregated artist-level metrics with Z-ORDER optimization for fast analytical queries

***

## Silver Layer — Star Schema

The silver layer models the nested Spotify API response into a proper relational star schema:

| Table | Type | Description |
|---|---|---|
| `fact_albums` | Fact | Core album release records |
| `dim_artists` | Dimension | Artist attributes |
| `dim_images` | Dimension | Album artwork metadata |
| `bridge_artists_albums` | Bridge | Many-to-many artist ↔ album |
| `bridge_images_albums` | Bridge | Many-to-many image ↔ album |

***

## Gold Layer — Aggregations

| Table | Description |
|---|---|
| `gold_artists` | Artist-level rollup metrics |
| `gold_artist_growth` | Year-over-year release growth using window functions |

***

## Key Engineering Decisions

**Incremental loads with Delta MERGE**  
The pipeline uses Delta Lake's `MERGE` operation so each run only processes new records rather than reprocessing the entire dataset.

**Data quality checks between layers**  
Referential integrity checks and null/missing value validations run between each layer transition to catch bad data early.

**Broadcast joins for small dimensions**  
Dimension tables are broadcast-joined to avoid unnecessary shuffles when joining against the large fact table.

**Z-ORDER optimization**  
Gold-layer tables are Z-ORDERed on frequently queried columns for faster analytical access patterns.

***

## Tech Stack

**Python · PySpark · Delta Lake · Spotify Web API · Docker · GitHub Actions**

***

<p align='justify'>
<a href="https://github.com/MLArchitect/pyspark-spotify-etl" target="_blank">View the code on GitHub</a>
</p>
