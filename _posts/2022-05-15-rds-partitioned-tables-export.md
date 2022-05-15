---
title: "Fun with exporting Postgres RDS partitioned tables to s3"
date: 2022-05-15
layout: post
---

## Rationale 

One of my recent tasks was refining a script which exports our RDS databases into s3. The concept was straightforward; we would utilize the daily system exports of our RDS instances, and export them to one of our s3 buckets as parquet files using boto3 python library. The IAM roles and s3 buckets were setup following the [AWS documentation](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_ExportSnapshot.html), including a KMS key to encrypt our exports, and the [boto3 function](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/rds.html#RDS.Client.start_export_task) for starting the export was also straightforward. The API has an optional list field `ExportOnly` where we could provide the list of databases or schemas or tables to include in the export. This is the fun point of this post. In the first iterations we decided to leave it empty and see what is being exported and experiment accordingly.

## Testing in staging 

We have a few databases hosting different data for the application's processing needs and we also have 2 environments (staging and production). We started with exporting our biggest staging RDS instance which does not include any partitioned tables. Everything went fine, export required a couple minutes (we also added a few more functions in our script to [monitor the export status](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/rds.html#RDS.Client.describe_export_tasks)). Also data looked good, tables were exported in different folders including their schema:

```
   ... /database_name/schema_name.first_table/first.parquet
											 /second.parquet
											   ....
					 /schema_name.second_table/first.parquet
										 	  /second.parquet
										 	  ...
					 /...
```

We then proceeded testing the export in another RDS instance where almost all tables are partitioned based on a tenant identifier. Hence these tables have naming scheme as such `table_name, table_name_1, table_name_2, ..., table_name_10000, table_name_default`. The staging instance had around 14k table partitions deriving from 7 parent tables, but not so much data in total. Export was equally quick with the previous one, structure was a little bit weird but expected:

```
   ... /database_name/schema_name.first_table_1/first.parquet
											   /second.parquet
											   ....
					 /schema_name.first_table_2/first.parquet
										 	   /second.parquet
										 	   ...
					 /schema_name.first_table_2000/first.parquet
										 	   	  /second.parquet
										 	   ...
					 /schema_name.first_table/first.parquet
										 	 /second.parquet
										 	   ...
```

This means that both the parent and partitioned tables were exported, with potentially duplicate data. A quick check using a Glue crawler and an Athena query verified the duplication. **For some reason AWS exports all table data into the parent table folder, as well as each partitioned table's data into its own folder.** We couldn't find any documentation explaining why, we just accepted the way it works and tried to adapt. We just added the list of tables we wanted to export in the `ExportOnly` argument of the `start_export_task` function. Next export iteration with this setting was also quick and now the folder structure was as if the database didn't have any partitioned tables. Next step was testing with our production instances.

## Testing in production

Testing in production required different IAM role, s3 bucket and KMS key. The first test using the RDS instance without partitioned tables was a total success, as it required around 30 minutes from start to end. More than 1TB of data were exported to s3 and tables' folders contained from 1 to a few thousand parquet files. Implementation-wise this was a big relief as we definitely needed this database export to be done on a daily basis for our internal data analytics needs. Its pricing is another issue and i will tackle it in another post in the future and we are currently trying to bypass the export solution using [Postgres' WAL-based](https://www.postgresql.org/docs/current/logical-replication.html) change data capture (CDC).

When we tried exporting the highly-partitioned production database we were surprised by the results. The database had 100k tables (5 parent tables with 20k partitions each and 2 unpartitioned tables) with around 1TB of data and export took almost 1 day. This was not a viable approach, as we wanted to setup some ETL workflows for each of the databases and having it running for 1+ day wouldn't make any sense. Without looking at the data we were expecting the export to be completed in a similar timeframe as the non-partitioned one, since we were just exporting the 7 parent tables. When we actually checked the folder structure, we were surprised to find out that each folder had a structure like below:
```
... /database_name/schema_name.first_table/1/first.parquet
											/second.parquet
											   ....
				  /schema_name.first_table/2/first.parquet
										 	/second.parquet
										 	   ...
				  /schema_name.first_table/20000/first.parquet
										 	   /second.parquet
										 	   ...

```

Notice the folders with the partition numbers. Hence the export task created around 100k folders with a few thousand small parquet files each (a couple kb each), which means a few hunded millions of parquet files. We assumed that the overhead was on s3 also considering [s3 limitations](https://docs.aws.amazon.com/AmazonS3/latest/userguide/optimizing-performance.html) and that's why the export was so slow. It wasn't clear why these partitioned-like folders were generated under the main folders which created the issue in the first place (remember that in staging we didn't see similar structure). Unfortunately there isn't any AWS documentation on the topic. In any case, we abandoned any further experimentation with exporting any of our highly-partitioned databases and we aim to implement it using CDC.

## Conclusion

Exporting RDS snapshots to s3 is an expensive operation, and its application to a highly-partitioned database is almost non-viable. I wish AWS provided some proper documentation on how the whole operation works and how it could be optimized, 