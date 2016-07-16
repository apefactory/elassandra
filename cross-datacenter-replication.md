# Analyzing Lastfm dataset with Elassandra on 2 datacenters

To demonstrate the ability to provide elaticsearch cross datacenter replication with elassandra, we have build a 4 nodes elassandra cluster,
2 nodes on a europe-west1 google datacenter, and 2 nodes on the us-central datacenter.

<img alt="Google cloud console screenshot" src="https://github.com/strapdata/blog.elassandra.io/blob/gh-pages/assets/images/gce-console1.png" width="900">

Then we have created a lastfm keyspace, replicated with a replication factor of 2 in each datacenters.

```
elassandra-eu-01:/opt/cqlinject-0.1/test$ cqlsh
Connected to Test Cluster at elassandra-eu-01:9042.
[cqlsh 5.0.1 | Cassandra 2.2.6 | CQL spec 3.3.1 | Native protocol v4]
Use HELP for help.
cqlsh> desc KEYSPACE lastfm ;
CREATE KEYSPACE lastfm WITH replication = {'class': 'NetworkTopologyStrategy', 'europe-west1': '2', 'us-central1':'2'}  AND durable_writes = true;
CREATE TABLE lastfm.playlist (
    user_id text,
    datetime timestamp,
    age int,
    artist_id text,
    artist_name text,
    country text,
    gender text,
    registred timestamp,
    song text,
    song_id text,
    PRIMARY KEY ((user_id, datetime))
    ) WITH bloom_filter_fp_chance = 0.01
    AND caching = '{"keys":"ALL", "rows_per_partition":"NONE"}'
    AND comment = ''
    AND compaction = {'class': 'org.apache.cassandra.db.compaction.SizeTieredCompactionStrategy'}
    AND compression = {'sstable_compression': 'org.apache.cassandra.io.compress.LZ4Compressor'}
    AND dclocal_read_repair_chance = 0.1
    AND default_time_to_live = 0
    AND gc_grace_seconds = 864000
    AND max_index_interval = 2048
    AND memtable_flush_period_in_ms = 0
    AND min_index_interval = 128
    AND read_repair_chance = 0.0
    AND speculative_retry = '99.0PERCENTILE';
```

Using a simple cassandra injector, we have inserted 39M rows from the [lasterfm dataset-1K](http://www.dtic.upf.edu/~ocelma/MusicRecommendationDataset/lastfm-1K.html).

```
elassandra-eu-01:/opt/cqlinject-0.1/test$ ./load.sh
SLF4J: Failed to load class "org.slf4j.impl.StaticLoggerBinder".
SLF4J: Defaulting to no-operation (NOP) logger implementation
SLF4J: See http://www.slf4j.org/codes.html#StaticLoggerBinder for further details.
Connected to cluster: Test Cluster
Datatacenter: europe-west1; Host: elassandra-eu-01/10.132.0.2; Rack: b
Datatacenter: europe-west1; Host: /10.132.0.4; Rack: c
Datatacenter: us-central1; Host: /10.128.0.2; Rack: b
Datatacenter: us-central1; Host: /10.128.0.3; Rack: c
Loading file userid-profile.tsv
992 profiles loaded.
Reading file userid-timestamp-artid-artname-traid-traname.tsv separator='       '
INSERT INTO "lastfm"."playlist" (user_id,datetime,artist_id,artist_name,song_id,song,gender,age,country,registred) VALUES (?,?,?,?,?,?,?,?,?,?)
[user_000001, Mon May 04 23:08:57 UTC 2009, f1b1cf71-bd35-4e99-8624-24a6e15f133a, Deep Dish, , Fuck Me Im Famous (
Pacha Ibiza)-09-28-2007, m, 45, Japan, Thu Feb 23 00:00:00 UTC 2006]
```

## Indexing with Elasticsearch

After sevral minutes, we have created a set of per-year partitioned elasticsearch indices using the script below. All columns of the lastfl.playlist cassandra table were indexed with the default mapping, except the song column where we choose to anaylse the text content.

```
function create_index() {
   curl -XPUT "http://$HOSTNAME:9200/lastfm_$1" -d '{
        "settings":{
                "keyspace":"lastfm",
                "index.partition_function":"year lastfm_{0,date,yyyy} datetime" },
        "mappings":{
            "playlist":{
                "discover":"(?!song).*",
                "properties": {
                        "song":{"type":"string","index":"analyzed","cql_collection":"singleton"}
                }
            }
        }
   }'
}
for y in 2005 2006 2007 2008 2009 2010 2011 2012 2013
do
   create_index $y
done
```
As soon these elasticsearch indices were created, cassandra secondary indices start to index existing and inserted data, involving a CPU overload to produce the underling lucene files. 

<img alt="Visual VM of elassandra-eu-01" 
src="https://github.com/strapdata/blog.elassandra.io/blob/gh-pages/assets/images/visualvm1.png" width="900">

Obviously, in the same time, the injector write throughput has decreased. 

<img alt="Injector cassandra driver metrics" src="https://github.com/strapdata/blog.elassandra.io/blob/gh-pages/assets/images/cqlinject-when-indexing.png" width="900">

As we have created elasticsearch indices sevral minutes after the injector start to insert data, cassandra lauch a compaction the build secondary index on the existing data, while indexing the new one. This explains why write throughput significantly decreased when indices where created.

```
$ nodetool compactionstats
pending tasks: 10
                                     id         compaction type   keyspace      table   completed      total    unit   progress
   a6b5bd00-42de-11e6-b23d-4b686713c22c   Secondary index build     lastfm   playlist     3468894   94640715   bytes      3,67%
   a9496030-42de-11e6-b23d-4b686713c22c   Secondary index build     lastfm   playlist      206700   96656079   bytes      0,21%
Active compaction remaining time :   0h00m00s
```

While the injector was inserting at a rate of 4000 documents/s on the europe-west1 datacenter, we started a kibana server on elassandra-us-02 in the us-central1 datacenter. As shown, a dashboard on
lastfm data provide 

<img alt="Kibana report while injecting" src="https://github.com/strapdata/blog.elassandra.io/blob/gh-pages/assets/images/kibana-report-while-injecting.png" width="900">

When the injector finished to inject almost 19M rows, CPU load load began to decrease on elassandra nodes, but remaing compactions were still consuming about 60% CPU.

```
$ bin/nodetool -h 104.197.104.224 compactionstats
objc[5394]: Class JavaLaunchHelper is implemented in both /Library/Java/JavaVirtualMachines/jdk1.8.0_65.jdk/Contents/Home/bin/java and /Library/Java/JavaVirtualMachines/jdk1.8.0_65.jdk/Contents/Home/jre/lib/libinstrument.dylib. One of the two will be used. Which one is undefined.
pending tasks: 17
                                     id         compaction type   keyspace      table    completed        total    unit   progress
   3fdfb020-42ef-11e6-a4f6-33f8af070846              Compaction     lastfm   playlist   4585897305   4612845649   bytes     99,42%
   b8b60b90-42de-11e6-a4f6-33f8af070846   Secondary index build     lastfm   playlist     73053591     98896356   bytes     73,87%
Active compaction remaining time :   0h00m01s
```

<img alt="VisualVM after the injector ends" src="https://github.com/strapdata/blog.elassandra.io/blob/gh-pages/assets/images/visualvm4.png" width="900">

At the end, we had our 18.7M documents available on the remote datacenter, ready for visualization in kibana.

<img alt="Kibana report at the end" src="https://github.com/strapdata/blog.elassandra.io/blob/gh-pages/assets/images/kibana-end-report.png" width="900">

Looking at cluster state with the [elasticHQ](http://www.elastichq.org/) plugin, you can notice that the total number of document is twice the one available in kibana. This is because all elassandra nodes are primary, so replicated data are indexed twice in a primary shards. 

<img alt="ElasticHQ erroneous total document" src="https://github.com/strapdata/blog.elassandra.io/blob/gh-pages/assets/images/elastichq.png" width="900">

Finally, in each datacenter, we have 8Gb of cassandra data and 6Gb of elasticsearch index files located in /var/lib/cassandra/data/elasticsearch.data.

```
Datacenter: europe-west1
========================
Status=Up/Down
|/ State=Normal/Leaving/Joining/Moving
--  Address     Load       Tokens       Owns    Host ID                               Rack
UN  10.132.0.4  4.13 GB    32           ?       1784411f-f7e3-48d5-a51a-ed8c20cd67ef  c
UN  10.132.0.2  3.83 GB    32           ?       6fbeeef7-df98-439a-9976-01eae8e18b86  b
Datacenter: us-central1
=======================
Status=Up/Down
|/ State=Normal/Leaving/Joining/Moving
--  Address     Load       Tokens       Owns    Host ID                               Rack
UN  10.128.0.2  3.86 GB    32           ?       289ca45a-1b5b-4847-a241-ffbc59c5b32e  b
UN  10.128.0.3  3.86 GB    32           ?       ef03f183-2982-4a1d-b636-d57e811d5a90  c
```
