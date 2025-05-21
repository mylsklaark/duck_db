# Building a Data Pipeline with DuckDB

This project is a copy of Robin Moffat's work, found at: https://rmoff.net/2025/03/20/building-a-data-pipeline-with-duckdb/. 

I wanted to practice ETL and data engineering with DuckDB, all of which he uses in his example. I will then use this as a basis for my own project. Some of the below is re-written from Robin's blog. I've added my own notes to clarify things that are new to me/help information enter my brain.

The project uses the Environment Agency's 'Real Time flood-monitoring API', found here: https://environment.data.gov.uk/flood-monitoring/doc/reference. Said data is readings, providing information about measures such as rainfall and river levels. They are reported from a variety of stations around the UK.

## Overview

### The data

Regarding the data, Robin notes:

_At the heart of the data are readings, providing information about measures such as rainfall and river levels. These are reported from a variety of stations across the U.K._

| Readings       | Measures       | Stations       |
|-----------------|----------------|----------------|
| Fact data (e.g. water level) grain: timestamp/measure/station  | Dimension/reference data (e.g. units, type of measurement, update frequency)  | Dimension/reference data (e.g location, town, area, lat/long)  |
| New readings every 15 minutes but sometimes less frequently  | Full dataset returned on each pull. Delta is negligible; only when measures are added or removed from stations  | Full dataset returned on each pull. Delta is negligible; only when stations are added or removed  |
| Per time and measure  | One or more readings and applies to only one station  | Each station has one or more measures  |

DuckDB will bw used throughout this project. It can:

- read data from [REST APIs](https://www.ibm.com/think/topics/rest-apis#:~:text=A%20REST%20API%20is%20an,to%20connect%20distributed%20hypermedia%20systems.)
- process data
- store data
- be queried with a range of visualisation tools

I'll use a local installation on my Mac, referercing specific commands here to help me when I revist the project.

In terms of a data warehousing or modelling context, we will be applying Slowly Changing Dimension Type 1 (SCD Type 1). This means that:

- the dimension data is being overwritten each time with no version or history kept
- when a station is updated (for example, its name or location changes), past readings will reflect the latest station information, even if it didn't apply at the time
- if a station is removed, and the dimension record is deleted, the older readings that referenced it may now have no matching dimension. This could potentially cause integrity or join issues in reporting

The real-world implications of this are:

If a station named 'CherwellsAmazingWaterStation' changed its name to 'CherwellsEvenMoreAmazingWaterStation', in SCD Type 1:

- all past readings originally associated with 'CherwellsAmazingWaterStation' will now appear as if they were from 'CherwellsMoreAmazingWaterStation'
- there is no way to trace that the station used to have a different name
- if the station is deleted from the dimension table old readings might become 'orphans' - fact records with no corresponding dimension data

### The plan

- Periodic ingest to staging tables
- Rebuild dimension tables
- Update fact table
- Fact/Dim join
- Analyse data

> Run the following command in the terminal to start the DuckDB UI:
>
> ```bash
> duckdb env-agency.duckdb -ui
> ```

## Extract

See the DuckDB notebook for extraction.

## Transform

### Keys

The staging tables have no keys defined because they are intended to serve as a raw data ingestion layer. Staging is where we bring in the source data in its original form, including any inconsistencies and/or anomalies. While a station is not expected to have duplicate entries, the staging layer does not enforce this assumption to ensure all source data is captured for further processing and validation. Therefore, it's important to code defensively so that, while we will accept anything into the staging area, we won't allow all of it to enter the pipeline. Or as Robin puts it:

> " ...we won't blindly propgate crap through the rest of the pipeline".

Words to live by.

### Readings > Measures

The readings table has a column called measure. This contains a full URL identifying the measure for each reading. For example:

```
http://environment.data.gov.uk/flood-monitoring/id/measures/5312TH-level-stage-i-15_min-mASD
```

In the measures table, the @id column contains the exact same URL as its unique identifier. However, the measures table also has a notation column, which contains only the final part of the URL (the unique code). It looks like this:

```
5312TH-level-stage-i-15_min-mASD
```

Therefore, to make joining these tables easier and more readable, we will pre-process he readings.measure column to remove the URL prefix. This will leave only the unique code. This way, we can join readings and measures using the shortened value (notation). This will make our SQL queries simpler and easier to debug, since we won't be working with columns of long text.

### Measures > Stations

The station for which a reading was taken is found via the measure, since measures are unique to a station. On measures, the station column is the foreign key:

```
http://environment.data.gov.uk/flood-monitoring/id/stations/SP50_72
```

Again, the URL prefix (http://environment.data.gov.uk/flood-monitoring/id/stations/) is repeated, so we'll strip it out.

Note that station (minus the URL prefix) and the stationReference are not always the same. The latter is not unique.

## Dimension tables

Building the dimension tables is simple enough. With a CREATE or REPLACE we tell DuckDB to create the table and, if it exists already, delete it and create a new one.

### Measures

We'll drop a couple of fields:

- we don't need @id
- latestReading holds fact data that we're getting from elsewhere, so there's no need to duplicate it here

We'll also transform the foreign key to strip the URL prefix making it easier to work with.

```
CREATE OR REPLACE TABLE measures AS
    SELECT *
            EXCLUDE ("@id", latestReading)
            REPLACE(
                REGEXP_REPLACE(station,
                        'http://environment\.data\.gov\.uk/flood-monitoring/id/stations/',
                        '') AS station
            )
    FROM measures_stg;
```

This uses two helpful clauses in DuckDB, namely EXCLUDE and REPLACE.

With EXCLUDE we're taking advantage of SELECT * to bring in all columns from the source table. This saves typing but also means new columns added to the source table will propogate automatically, with the exception of the ones that we don't want.

REPLACE is an elegant way of providing a different version of the same column. Since we want to retain the station column but just trim the prefix, this is a great way of doing it without moving its position in the column list. The other option would have been to EXCLUDE it too, and then add it on to the column list.