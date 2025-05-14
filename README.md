### Building a Data Pipeline with DuckDB

This project is a copy of Robin Moffat's work, found at: https://rmoff.net/2025/03/20/building-a-data-pipeline-with-duckdb/. I wanted to practice ETL and  data engineering with DuckDB, all of which he uses in his example. I will then use this as a basis for my own project. Some of the below is re-written from Robin's blog. I've added my own notes to clarify things that are new to me/help information enter my brain.

The project uses the Environment Agency's 'Real Time flood-monitoring API', found here: https://environment.data.gov.uk/flood-monitoring/doc/reference. Said data is readings, providing information about measures such as rainfall and river levels. They are reported from a variety of stations around the UK.

### Overview

## The data

Regarding the data, Robin notes:

_At the heart of the data are readings, providing information about measures such as rainfall and river levels. These are reported from a variety of stations across the U.K._

| Readings       | Measures       | Stations       |
|-----------------|----------------|----------------|
| Fact data (e.g. water level) grain: timestamp/measure/station  | Dimension/reference data (e.g. units, type of measurement, update frequency)  | Dimension/reference data (e.g location, town, area, lat/long)  |
| New readings every 15 minutes but sometimes less frequently  | Full dataset returned on each pull. Delta is negligible; only when measures are added or removed from stations  | Full dataset returned on each pull. Delta is negligible; only when stations are added or removed  |
| Per time and measure  | One or more readings and applies to only one station  | Each station has one or more measures  |

DuckDB will bw used throughout this project. It can:

- read data from REST APIs
- process data
- store data
- be queried with a range of visualisation tools

In terms of a data warehousing or modelling context, we will be applying Slowly Changing Dimension Type 1 (SCD Type 1). This means that:

- the dimension data is being overwritten each time with no version or history kept
- when a station is updated (for example, its name or location changes), past readings will reflect the latest station information, even if it didn't apply at the time
- if a station is removed, and the dimension record is deleted, the older readings that referenced it may now have no matching dimension. This could potentially cause integrity or join issues in reporting

The real-world implications of this are:

If a station named 'CherwellsAmazingWaterStation' changed its name to 'CherwellsMoreAmazingWaterStation', in SCD Type 1:

- all past readings originally associated with 'CherwellsAmazingWaterStation' will now appear as if they were from 'CherwellsMoreAmazingWaterStation'
- there is no way to trace that the station used to have a different name
- if the station is deleted from the dimension table old readings might become 'orphans' - fact records with no corresponding dimension data

## The plan

Periodic ingest to staging tables

Rebuild dimension tables

Update fact table