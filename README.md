## Detecting::/128 in traceroute data 

In this tutorial, we explain the steps to build a SQL query to find  `::` in the traceroute dataset. You can find the query at the end of the document. In the following section, we give step by step breakdown of the query. 

#### 1. Finding traceroutes with `::` in hops
We start by querying the traceroute table (`ripencc-atlas.measurements.traceroute`) to find all the hops. It may be helpful to have a glance at the [traceroute schema table](https://github.com/RIPE-NCC/ripe-atlas-bigquery/blob/fea4b68f251bd4f72e482cfc3803aaa98de4abab/docs/measurements_traceroute.md). We then join it with Probes table to get metadata about RIPE Atlas probes. 

```sql
with traces as( SELECT
     prb_metadata.prb_id as probe_id,prb_metadata.asn_v6 as probe_asn_v6, prb_metadata.country_code as probe_country_code,
     traces.src_addr as src_addr,traces.dst_addr as dst_addr, hops,
   FROM
    `ripencc-atlas.measurements.traceroute` as traces, UNNEST (hops) AS hop 
   join `ripencc-atlas.probes.probes` as prb_metadata on traces.prb_id= prb_metadata.prb_id
  WHERE
    DATE(start_time) = "2022-04-20"
    AND hop.hop_addr='::'
    AND traces.dst_addr!='::')
```
Note that we have to unnest the hops since they are NESTED schema to filter results on the hops



#### 2. Get unique pairs
Traceroute sends three packets for each hop. In our dataset, we have three responses from the same hop. Since we are only interested in topology, we discard the additional information andonly keep unique hops. 

```sql
unique as 
(
   select
      * REPLACE( (
      SELECT
         ARRAY_AGG(STRUCT(hop_addr, hop)) --struct
      FROM
         (
            SELECT DISTINCT
               hop_addr,
               hop 
            FROM
               UNNEST(hops) c 
         )
) AS hops ) 
      FROM
         traces
)
```
#### 3. Correct the order of hops
Finally, we correct the order of the hops and select the final dataset for further analysis. 
```sql
select
   probe_id,
   probe_asn_v6,
   probe_country_code,
   src_addr,
   dst_addr,
   ARRAY(
   SELECT
      STRUCT (hop_addr, hop) 
   FROM
      UNNEST(hops) AS hops 
   ORDER BY
      hop) AS hops 
   from
      unique
```



## Complete Query




```sql

WITH traces
AS (
	SELECT prb_metadata.prb_id AS probe_id
		,prb_metadata.asn_v6 AS probe_asn_v6
		,prb_metadata.country_code AS probe_country_code
		,traces.src_addr AS src_addr
		,traces.dst_addr AS dst_addr
		,hops
		,
	FROM `ripencc-atlas.measurements.traceroute` AS traces
		,UNNEST(hops) AS hop
	JOIN `ripencc-atlas.probes.probes` AS prb_metadata ON traces.prb_id = prb_metadata.prb_id
	WHERE DATE (start_time) = "2022-04-20"
		AND hop.hop_addr = '::'
		AND traces.dst_addr != '::'
	)
	,UNIQUE
AS (
	SELECT * REPLACE((
				SELECT ARRAY_AGG(STRUCT(hop_addr, hop))
				FROM (
					SELECT DISTINCT hop_addr
						,hop
					FROM UNNEST(hops) c
					)
				) AS hops)
	FROM traces
	)


-- the final query
SELECT probe_id
	,probe_asn_v6
	,probe_country_code
	,src_addr
	,dst_addr
	,ARRAY(SELECT STRUCT(hop_addr, hop) FROM UNNEST(hops) AS hops ORDER BY hop) AS hops
FROM UNIQUE
```
