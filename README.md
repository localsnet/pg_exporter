# PG Exporter

[Prometheus](https://prometheus.io/) [exporter](https://prometheus.io/docs/instrumenting/exporters/) for [PostgreSQL](https://www.postgresql.org) metrics. Latest binaries can be found on [release](https://github.com/Vonng/pg_exporter/releases) page. 

supporting PostgreSQL 10+ & Pgbouncer 1.9+。



## Features

* Support both Postgres & Pgbouncer (auto-switch when target database name is pgbouncer)
* Fine-grained execution control (cluster/database level, primary/standby only, tags, etc...)
* Flexible: completely customizable queries & metrics
* Configurable caching policy & query timeout
* Including many internal metrics, built-in self-monitoring
* Dynamic query planning, brings more fine-grained contronl on execution policy
* Auto discovery multi database in the same cluster (TBD) 
* Tested in real world production environment (200+ Nodes)



## Quick Start

To run this exporter, you will need two things

* **Where** to scrape:  A postgres or pgbouncer URL given via `PG_EXPORTER_URL`  or `--url`
* **What** to scrape: A path to config file or directory, by default `./pg_exporter.yaml`

```bash
export PG_EXPORTER_URL='postgres://postgres:password@localhost:5432/postgres'
export PG_EXPORTER_CONFIG='/path/to/conf/file/or/dir'
pg_exporter
```

`pg_exporter` only built-in with 3 metrics: `pg_up`,`pg_version` , and  `pg_in_recovery`. **All other metrics are defined in configuration files** . You cound use pre-defined configuration file: [`pg_exporter.yaml`](pg_exporter.yaml) or use separated metric query in [conf](https://github.com/Vonng/pg_exporter/tree/master/conf)  dir.

Parameter could be given via command line arguments or environment variables. 

```bash
usage: pg_exporter [<flags>]

Flags:
  --help                         Show context-sensitive help
  --url=URL                      postgres connect url
  --config="./pg_exporter.yaml"  Path to config files
  --label=""                     Comma separated list label=value pair
  --tag=""                       Comma separated list of server tag
  --disable-cache                force not using cache
  --auto-discovery               scrape all database for given cluster
  --exclude-database="postgres,template0,template1"
                                 excluded databases when enabling auto-discovery
  --namespace=""                 prefix of built-in metrics (pg|pgbouncer) by default
  --fail-fast                    fail fast instead of waiting during start-up
  --web.listen-address=":9630"   prometheus web server listen address
  --web.telemetry-path="/metrics"
                                 Path under which to expose metrics.
  --dry-run                      dry run and explain raw conf
  --explain                      explain server planned queries
  --version                      Show application version.
  --log.level="info"             Only log messages with the given severity or above.
  --log.format="logger:stderr"   Set the log target and format. 
```



## Build

```
go build
```

To build a static stand alone binary for docker scratch

```bash
CGO_ENABLED=0 GOOS=linux go build -a -ldflags '-extldflags "-static"' -o pg_exporter
```

To build a docker image, using:

```
docker build -t pg_exporter .
```

Or just [download](https://github.com/Vonng/pg_exporter/releases) latest prebuilt binaries on [release](https://github.com/Vonng/pg_exporter/releases) page



## Run

```bash
usage: pg_exporter [<flags>]

Flags:
  --help                         Show context-sensitive help
  --url=URL                      postgres connect url
  --config="./pg_exporter.yaml"  Path to config files
  --label=""                     Comma separated list label=value pair
  --tag=""                       Comma separated list of server tag
  --disable-cache                force not using cache
  --auto-discovery               scrape all database for given cluster
  --exclude-database="postgres,template0,template1"
                                 excluded databases when enabling auto-discovery
  --namespace=""                 prefix of built-in metrics (pg|pgbouncer) by default
  --fail-fast                    fail fast instead of waiting during start-up
  --web.listen-address=":9630"   prometheus web server listen address
  --web.telemetry-path="/metrics"
                                 Path under which to expose metrics.
  --dry-run                      dry run and explain raw conf
  --explain                      explain server planned queries
  --version                      Show application version.
  --log.level="info"             Only log messages with the given severity or above.
  --log.format="logger:stderr"   Set the log target and format. 

```



## Config

Config files are using YAML format, there are lots of examples in the [conf](https://github.com/Vonng/pg_exporter/tree/master/conf) dir. and here is a [sample](conf/100-doc.txt) config.

```yaml
#-------------------------------------------------------------#
# Metric Query Example
#-------------------------------------------------------------#

#  pg_primary_only:       <---- Branch name, distinguish different branch of a metric query
#    name: pg             <---- actual Query name, used as metric prefix, will set to branch if not provide
#    desc: PostgreSQL basic information (on primary)                    <---- query description
#    query: |                                                           <---- query string
#
#      SELECT extract(EPOCH FROM CURRENT_TIMESTAMP)                  AS timestamp,
#             pg_current_wal_lsn() - '0/0'                           AS lsn,
#             pg_current_wal_insert_lsn() - '0/0'                    AS insert_lsn,
#             pg_current_wal_lsn() - '0/0'                           AS write_lsn,
#             pg_current_wal_flush_lsn() - '0/0'                     AS flush_lsn,
#             extract(EPOCH FROM now() - pg_postmaster_start_time()) AS uptime,
#             extract(EPOCH FROM now() - pg_conf_load_time())        AS conf_reload_time,
#             pg_is_in_backup()                                      AS is_in_backup,
#             extract(EPOCH FROM now() - pg_backup_start_time())     AS backup_time;
#
#                                <---- following field are [OPTIONAL], control execution policy
#    ttl: 10                     <---- cache ttl: how long will exporter cache it's result. set to 0 to disable cache
#    timeout: 0.1                <---- timeout: in seconds, query execeed this will be canceled. default is 0.1, set to -1 to disable timeout
#    min_version: 100000         <---- minimal supported version in server version number format, e.g  120001 = 12.1, 090601 = 9.6.1
#    max_version: 130000         <---- maximal supported version in server version number format, boundary not include
#    fatal: false                <---- if query marked fatal fail, this scrape will abort immidiately
#
#    tags: [cluster, primary]    <---- tags consist of one or more string, which could be:
#                                        * 'cluster' marks this query as cluster level, so it will only execute once for same PostgreSQL Server
#                                        * 'primary' marks this query can only run on a master instance (will not execute if pg_is_in_recovery())
#                                        * 'standby' marks this query can only run on a recoverying instance (will execute if pg_is_in_recovery())
#                                        * 'dbname:<dbname>' means this query will only execute on database with name '<dbname>'
#                                        * 'extension:<extname>' means this query will only execute when extension '<extname>' is installed
#                                        * 'schema:<nspname>' means this query will only execute when schema '<nspname>' exist
#                                        * 'not:<negtag>' means this query will only execute when exporter is launch without tag '<negtag>'
#                                        * '<tag>' means this query will only execute when exporter is launch with tag '<tag>'
#                                           (tag could not be cluster,primary,standby or have special prefix)
#
#
#    metrics:                    <---- this is a list of returned columns, each column must have a name, usage, could have an alias and description
#      - timestamp:              <---- this is column name, should be exactly same as returned column name
#          rename: ts            <---- rename is optional, will use this alias instead of column name
#          usage: GAUGE          <---- usage could be
#                                        * DISCARD: completely ignore this field
#                                        * LABEL: use columnName:columnValue as a label in result
#                                        * GAUGE: use this column as a metric, which is '<query.name>_<column.name>{<labels>} column.value'
#                                        * COUNTER: same as GAUGE, except it is a counter.
#
#          description: database current timestamp
#      - lsn:
#          usage: COUNTER
#          description: log sequence number, current write location (on primary)
#      - insert_lsn:
#          usage: COUNTER
#          description: primary only, location of current wal inserting
#      - write_lsn:
#          usage: COUNTER
#          description: primary only, location of current wal writing
#      - flush_lsn:
#          usage: COUNTER
#          description: primary only, location of current wal syncing
#      - uptime:
#          usage: GAUGE
#          description: seconds since postmaster start
#      - conf_reload_time:
#          usage: GAUGE
#          description: seconds since last configuration reload
#      - is_in_backup:
#          usage: GAUGE
#          description: 1 if backup is in progress
#      - backup_time:
#          usage: GAUGE
#          description: seconds since current backup start. null if don't have one



```



## FAQ



## About

Author：Vonng ([fengruohang@outlook.com](mailto:fengruohang@outlook.com))

License：BSD

