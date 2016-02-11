---
layout: post
title: "Compressing Redshift tables"
excerpt: "Reducing space reduces cost and data IO/net, leading to faster queries"
tags: [redshift, aws, databases, ops]
comments: true
---

> *TL;DR* RDS downtime, scheduled or not, is reduced to around 2 minutes using an 
automatic failover mechanism available for multi-AZ deployments. 
At times, replica instances attached to the master node, if any, can loose the
master after failover: contacting AWS support would solve the issue (they
ackwnoledge it).
It is important to test all these mechanisms before RDS instance modification, 
in order to be able supervise the fail over procedure and accurately predict production downtime.
Using a single-AZ deployment, outage due to resizing can last much longer (~ XX minutes).
TODO: test restoring from snapshot or modifying up and down once.




PROCEDURE:

- Stop/Upload/Import collectors
- Get line numbers (use psql redshift admin)
- Do Backup redshift, always a good practice
- Unload All events table
- Truncate Events table
- VACUUM events (immediate)
- COPY command
- Check line numbers are ok
- VACUUM events (immediate)
- ANALYZE events (2m:55s)
- Start collectors




### WHY
Compression is a column-level operation that reduces the size of data when it is
stored. Compression conserves storage space and reduces the size of data that is
read from storage, which reduces the amount of disk I/O and therefore improves
query performance. 
Also, compression optimizes usage of disk space, and therefore cost.

### WHAT
Our production redshift cluster does not compress/encode table columns in
general (see enc column):

```
 schema |                             table                             |
tableid |  distkey  |  skew  |     sortkey     | #sks |    rows    | mbytes |
enc | pct_of_total | pct_stats_off | pct_unsorted                                                                           
--------+---------------------------------------------------------------+---------+-----------+--------+-----------------+------+------------+--------+-----+--------------+---------------+--------------
 public | events                                                        |
108502 | EVEN      | 1.0000 |                 |    0 | 2204642391 | 299661 | N
|        39.29 |          0.00 |
 public | sessions                                                      |
108495 | EVEN      | 1.0000 |                 |    0 |  126003060 |  18808 | N
|         2.46 |          0.00 |
 public | agg_revenue_forecast                                          |
193347 | EVEN      | 1.0000 |                 |    0 |   57818726 |  10936 | N
|         1.43 |          0.00 |
 public | monetizations                                                 |
108632 | EVEN      | 1.0000 |                 |    0 |   35294395 |   7600 | N
|         0.99 |          0.00 |
 public | transactions                                                  |
108465 | EVEN      | 1.0021 |                 |    0 |   35795089 |   7376 | N
|         0.96 |          0.00 |
 public | sponsorships                                                  |
108716 | user_id   | 1.0126 | signed_at       |    1 |   17493697 |   1903 | N
|         0.24 |          0.00 |         0.00
 public | agg_world_tours                                               |
142595 | EVEN      | 1.0000 |                 |    0 |   14405025 |   1888 | N
|         0.24 |          0.00 |
 public | users                                                         |
108391 | id        | 1.0000 | id              |    1 |    9259314 |   1888 | Y
|         0.24 |          0.00 |         0.00
 public | agg_revenue_forecast_ios                                      |
142655 | EVEN      | 1.0000 |                 |    0 |    7566431 |   1688 | N
|         0.22 |          0.00 |
...
... other tables
...
```

There are many tables that are suitable for compression (enc column): the
pristine procedure would be to test some random insertions and calculate the
optimal compression algorithm for each table using the [ANALYZE COMPRESSION](
http://docs.aws.amazon.com/redshift/latest/dg/tutorial-tuning-tablescompression.html)
command for redshift, such as in
[here](https://www.periscopedata.com/blog/redshift-maintenance.html).
This is not possible as tables are already in production and their Data
Definition Language (DDL) statements already hard-coded in golden-collector
migration. Rewriting this part would be time expensive, error prone and probably
would require significative downtime.
I add here the optimal encoding for the biggest uncoded table for reference.

```
ANALYZE COMPRESSION events COMPROWS 10000000

TABLE     COLUMN   ENCODING ALG
events  id  lzo
events  event_type  lzo
events  user_id lzo
events  timestamp lzo
events  product_name  lzo
events  ingots  lzo
events  money lzo
events  player_id delta32k
events  injury_id lzo
events  training_id delta
events  transaction_id  lzo
events  user_ingots bytedict
events  user_money  mostly32
events  user_level_id delta
events  user_league_id  lzo
events  user_league_position  delta
events  user_local_cup_id delta32k
events  user_local_cup_position lzo
events  team_id lzo
events  world_id  lzo
events  device_id lzo
events  device_type lzo
events  author_id lzo
events  team_overall_score  lzo
events  tour_id lzo
events  tour_step_id  lzo
events  custom_team_id  lzo
```

An alternative method is to use [automatic compression]
(http://docs.aws.amazon.com/redshift/latest/dg/c_Loading_tables_auto_compress.html):
this requires to COPY data that were previously UNLOADed to S3 to *empty*
tables, as this would by default trigger automatic compression procedures (see
link for details), that would leave to a table with compressed columns.
Aim at compression of the first 7 biggest uncompressed tables from smallest to,
if possible, biggest events table.
Dynamic tables that are updated with a COPY from collector every 10 minutes
would require collector stop.

### HOW
  Tune UNLOAD and COPY commands to implement the flow, creating new compressed
tables. Then 
  ALTER table names to put compressed table into production. Finally, if
business guys say nothing (they i.e. did not notice anything) or with an
explicit check from them, DROP the old_ uncompressed tables.


#### Example:
  - Dump the table into S3 (parallel ON)

  ``` 
  unload('select * from agg_world_tours')
  to 's3://goldenmanager-bigdata/data/agg_world_tours_dump/agg_world_tours'
  credentials 'aws_access_key_id=GG; aws_secret_access_key=GG'
  gzip
  ```

  - Create an empty copy of the table using the DDL or CREATE TABLE import_table
    (LIKE table)
  ```
  copy import_agg_world_tours from
's3://goldenmanager-bigdata/data/agg_world_tours_dump/agg_'
  credentials 'aws_access_key_id=GG;aws_secret_access_key=GG' 
  COMPUPDATE ON
  GZIP
  ```
  - Rename 'old' encompressed table to old and the imported, compressed table to
    the production one
  ```
    ALTER TABLE agg_world_tours RENAME TO old_agg_world_tours;
    ALTER TABLE import_agg_world_tours RENAME TO agg_world_tours;
  ```

  - After everything works in production and business guys are happy, you can
    DROP the old uncompressed tables.
  ```
  DROP TABLE old_*
  ```

### Results

| table name | # records | start table size (MB) | UNLOAD  (t) | COPY (t)| enc
table size (MB)| 
|:--------:|:---------:|:---------:|:------------:|:--------:|:-------:|
|   agg_revenue_forecast_ios | 7,566,431|  1,688  |  ~16s  |  1m:43s  | 528 |
|   agg_world_tours |   14,405,025 | 1,888  |  ~14s  |  1m:27s  | 712 |
|   agg_revenue_forecast |57,818,726 | 10,936   |  1m:49s  |  4m:7s  |3,408 |
|   sponsorships | 17,754,972 | 1,911  |  ~15s  |  ~35s  |  865 |
|   transactions | 36,816,847 | 7,568  |  1m:09s  | 2m:20s   | 4073 |
|   monetizations  | 36,383,750  | 7,790  | 1m:33s | 2m:39s | 4,336 |
|   sessions | 129,048,295 | 19,288  | 4m:06s   |  4m:21s  | 11,264 |
|   events | 2,283,718,416 |  311,573  |  1h:03m  |  1h:16m  | 157,150 |

#### Events compressions: total redshift disk usage from 60% to 35%

- You can also see how long the export (UNLOAD) and import (COPY) lasted.
- The table was TRUNCATED after export to S3, as the import_* table procedure
  created problems with views, as it required the DROP of old_table, and with it
dependent objects would be deleted as well.
Aftert truncating and importing checked that count(*) was the same, i.e. perfect
import.
![compressing_events](https://cloud.githubusercontent.com/assets/7311157/12782234/08e629d4-ca7a-11e5-8314-e6a61d68b687.png)

The events columns have been encoded through automatic compression quite
differently then what suggested by ANALYZE (see above) 

![enc_with_automatic_compression](https://cloud.githubusercontent.com/assets/7311157/12784204/00f4a816-ca86-11e5-97f8-07dac60ce625.png)

