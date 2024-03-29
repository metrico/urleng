![visitor badge](https://img.shields.io/endpoint?url=https://y3kp3s.deta.dev)

# [Decentralized URL Engine for ClickHouse](https://urleng.com)

Cloudflare + Deta Worker serving a decentralized, stateful, serverless URL Engine platform for ClickHouse.

#### About

The serverless API enables separate ClickHouse servers to access external *distributed tables* hosted on the cloud and share any type of data between nodes and 3rd party applications in real-time through the native `URL` Engine and Functions available in ClickHouse.

![image](https://user-images.githubusercontent.com/1423657/179358754-92665ed1-c0f5-486f-9b94-e69144f8047a.png)

#### Serverless Platforms

The distributed URL engine platform is running on Cloudflare Workers and utilizing Deta Cloud as backend.

  - Cloudflare Worker NDJSON Proxy _(limited to 100k req/day)_ [*](https://developers.cloudflare.com/workers)
  - Cloudflare Worker API Handler _(limited to 100k req/day)_ [*](https://developers.cloudflare.com/workers)
  - Deta Micros + Base Storage _(unlimited requests, max 10k/inserts/query)_ [*](https://deta.sh)

--------

### Public Access Point

 - `https://urleng.com/{yoursupersecretkeygoeshere}`

--------

#### Usage
The following examples illustrates usage for distributed `INSERT` and `SELECT` statements

##### INSERT
```sql
:) INSERT INTO FUNCTION url('https://urleng.com/supersecret', JSONEachRow, 'key String, value UInt64') VALUES ('hello', 1), ('world', 2)
```
##### SELECT
```sql
:) SELECT * FROM url('https://urleng.com/supersecret', JSONEachRow);

┌─key───┬─value─┐
│ hello │     1 │
│ world │     2 │
└───────┴───────┘

2 rows in set. Elapsed: 0.126 sec. 
```

##### URL ENGINE
```sql
:) CREATE TABLE default.url_engine_distributed
   (
       `key` String,
       `value` UInt64,
   )
   ENGINE = URL('https://urleng.com/supersecret', 'JSONEachRow')
```
```sql
:) INSERT INTO url_engine_distributed VALUES ('hello', 1), ('world', 2)
:) SELECT * FROM url_engine_distributed

┌─key───┬─value─┐
│ hello │     1 │
│ world │     2 │
└───────┴───────┘

2 rows in set. Elapsed: 0.185 sec. 
```
###### Expiration
Items can be set to expire by including an `__expires` key in the object carrying a future Unix timestamp:
```sql
:) CREATE TABLE default.url_engine_expire
   (
       `key` String,
       `value` UInt64,
       `__expires` UInt64 DEFAULT toUInt64(toUnixTimestamp(now() + interval 24 hour))
   )
   ENGINE = URL('https://urleng.com/supersecret', 'JSONEachRow')
```

##### clickhouse-local
Get data into clickhouse-local with zero efforts:
```bash
clickhouse-local -q "select count() from url('https://urleng.com/supersecret', JSONEachRow)"
```

##### chdb
Get data using [chdb]([https://chdb.dev](https://chdb.fly.dev/?#U0VMRUNUICogZnJvbSB1cmwoJ2h0dHBzOi8vdXJsZW5nLmNvbS94eHgnLCBKU09ORWFjaFJvdywgJ2tleSBTdHJpbmcsIHZhbHVlIFVJbnQ2NCcpIExJTUlUIDEwOw==)) in-memory engine:
```bash
python -m chdb "SELECT * from url('https://urleng.com/xxx', JSONEachRow, 'key String, value UInt64');" Pretty
```

##### CURL

Insert and query data using curl or any other HTTP/S GET/POST capable client.

###### POST ndjson
```bash
curl -s -XPOST https://urleng.com/supersecret \
     -H 'Content-Type:application/x-ndjson' --data-binary @ndjson.txt
```
###### POST json
```bash
curl -X POST https://url-engine.metrico.in/supersecret \
     -H 'Content-Type: application/json' -d '[{"key":"curl","value":1}]'
```
###### GET
```bash
curl -X GET https://urleng.com/supersecret
```             

-----

#### Notes
- `INSERTS` are updates. Existing data will be replaced. Runtime limit ~1000 records per INSERT.
- `SELECT` pulls the full table, pending X-Header extensions to prefilter `WHERE` statements.
-  Because there is no sign-up, the URL `/path` is essentially a *password*, so pick something unique.

-------

#### Examples
- [URLENG Prometheus Metrics Storage + Exporter](https://github.com/metrico/distributed-clickhouse-prometheus)
- [URLENG Stand Alone](https://github.com/metrico/clickhouse-node-url-engine) example for self-hosted caches _(in memory)_
- [URLENG ClickHouse Fiddle](https://fiddle.clickhouse.com/2a71b82b-b7f8-4006-9e2e-01ee0519f0ca) example

(C) 2022 QXIP BV, for more info visit the [qxip metrico](https://metrico.in) repository
