# 元数据存储

The Metadata Storage is an external dependency of Apache Druid. Druid uses it to store
various metadata about the system, but not to store the actual data. There are
a number of tables used for various purposes described below.

Derby is the default metadata store for Druid, however, it is not suitable for production.
[MySQL](../development/extensions-core/mysql.md) and [PostgreSQL](../development/extensions-core/postgresql.md) are more production suitable metadata stores.

> The Metadata Storage stores the entire metadata which is essential for a Druid cluster to work.
> For production clusters, consider using MySQL or PostgreSQL instead of Derby.
> Also, it's highly recommended to set up a high availability environment
> because there is no way to restore if you lose any metadata.

## Using Derby

Add the following to your Druid configuration.

```properties
druid.metadata.storage.type=derby
druid.metadata.storage.connector.connectURI=jdbc:derby://localhost:1527//opt/var/druid_state/derby;create=true
```

## MySQL

See [mysql-metadata-storage extension documentation](../development/extensions-core/mysql.md).

## PostgreSQL

See [postgresql-metadata-storage](../development/extensions-core/postgresql.md).

## Adding custom dbcp properties

NOTE: These properties are not settable through the `druid.metadata.storage.connector.dbcp properties`: `username`, `password`, `connectURI`, `validationQuery`, `testOnBorrow`. These must be set through `druid.metadata.storage.connector` properties.

Example supported properties:

```properties
druid.metadata.storage.connector.dbcp.maxConnLifetimeMillis=1200000
druid.metadata.storage.connector.dbcp.defaultQueryTimeout=30000
```

See [BasicDataSource Configuration](https://commons.apache.org/proper/commons-dbcp/configuration) for full list.

## Metadata storage tables

### Segments table

This is dictated by the `druid.metadata.storage.tables.segments` property.

This table stores metadata about the segments that should be available in the system. (This set of segments is called
"used segments" elsewhere in the documentation and throughout the project.) The table is polled by the
[Coordinator](../design/coordinator.md) to determine the set of segments that should be available for querying in the
system. The table has two main functional columns, the other columns are for indexing purposes.

Value 1 in the `used` column means that the segment should be "used" by the cluster (i.e., it should be loaded and
available for requests). Value 0 means that the segment should not be loaded into the cluster. We do this as a means of
unloading segments from the cluster without actually removing their metadata (which allows for simpler rolling back if
that is ever an issue).

The `payload` column stores a JSON blob that has all of the metadata for the segment (some of the data stored in this payload is redundant with some of the columns in the table, that is intentional). This looks something like

```json
{
 "dataSource":"wikipedia",
 "interval":"2012-05-23T00:00:00.000Z/2012-05-24T00:00:00.000Z",
 "version":"2012-05-24T00:10:00.046Z",
 "loadSpec":{
    "type":"s3_zip",
    "bucket":"bucket_for_segment",
    "key":"path/to/segment/on/s3"
 },
 "dimensions":"comma-delimited-list-of-dimension-names",
 "metrics":"comma-delimited-list-of-metric-names",
 "shardSpec":{"type":"none"},
 "binaryVersion":9,
 "size":size_of_segment,
 "identifier":"wikipedia_2012-05-23T00:00:00.000Z_2012-05-24T00:00:00.000Z_2012-05-23T00:10:00.046Z"
}
```

Note that the format of this blob can and will change from time-to-time.

### Rule table

The rule table is used to store the various rules about where segments should
land. These rules are used by the [Coordinator](../design/coordinator.md)
  when making segment (re-)allocation decisions about the cluster.

### Config table

The config table is used to store runtime configuration objects. We do not have
many of these yet and we are not sure if we will keep this mechanism going
forward, but it is the beginnings of a method of changing some configuration
parameters across the cluster at runtime.

### Task-related tables

There are also a number of tables created and used by the [Overlord](../design/overlord.md) and [MiddleManager](../design/middlemanager.md) when managing tasks.

### Audit table

The Audit table is used to store the audit history for configuration changes
e.g rule changes done by [Coordinator](../design/coordinator.md) and other
config changes.

## Accessed by

The Metadata Storage is accessed only by:

1. Indexing Service Processes (if any)
2. Realtime Processes (if any)
3. Coordinator Processes

Thus you need to give permissions (e.g., in AWS Security Groups) only for these machines to access the Metadata storage.
