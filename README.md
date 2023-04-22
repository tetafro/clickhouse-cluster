# Clickhouse Cluster

Clickhouse cluster with 2 shards and 2 replicas built with docker-compose.

Not for production use.

## Run

Run single command, and it will copy configs for each node and
run clickhouse cluster `company_cluster` with docker-compose
```sh
make config up
```

Containers will be available in docker network `172.23.0.0/24`

| Container    | Address
| ------------ | -------
| zookeeper    | 172.23.0.10
| clickhouse01 | 172.23.0.11
| clickhouse02 | 172.23.0.12
| clickhouse03 | 172.23.0.13
| clickhouse04 | 172.23.0.14

## Profiles

- `default` - no password
- `admin` - password `123`

## Test it

Login to clickhouse01 console (first node's ports are mapped to localhost)
```sh
clickhouse-client -h localhost
```

Or open `clickhouse-client` inside any container
```sh
docker exec -it clickhouse01 clickhouse-client -h localhost
```

Create a test database and table (sharded and replicated)
```sql
CREATE DATABASE company_db ON CLUSTER 'company_cluster';

CREATE TABLE company_db.events ON CLUSTER 'company_cluster' (
    time DateTime,
    uid  Int64,
    type LowCardinality(String)
)
ENGINE = ReplicatedMergeTree('/clickhouse/tables/{cluster}/{shard}/events', '{replica}')
PARTITION BY toDate(time)
ORDER BY (uid);

CREATE TABLE company_db.events_distr ON CLUSTER 'company_cluster' AS company_db.events
ENGINE = Distributed('company_cluster', company_db, events, uid);
```

Load some data
```sql
INSERT INTO company_db.events_distr VALUES
    ('2020-01-01 10:00:00', 100, 'view'),
    ('2020-01-01 10:05:00', 101, 'view'),
    ('2020-01-01 11:00:00', 100, 'contact'),
    ('2020-01-01 12:10:00', 101, 'view'),
    ('2020-01-02 08:10:00', 100, 'view'),
    ('2020-01-03 13:00:00', 103, 'view');
```

Check data from the current shard
```sql
SELECT * FROM company_db.events;
```

Check data from all cluster
```sql
SELECT * FROM company_db.events_distr;
```

## Add more nodes

If you need more Clickhouse nodes, add them like this:

1. Add replicas/shards to `config.xml` to the block `company/remote_servers/company_cluster`.
1. Add nodes to `docker-compose.yml`.
1. Add nodes in `Makefile` in `config` target.

## Start, stop

Start/stop the cluster without removing containers
```sh
make start
make stop
```

## Teardown

Stop and remove containers
```sh
make down
```
