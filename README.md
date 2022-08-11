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
```
:) INSERT INTO FUNCTION url('https://urleng.com/supersecret', JSONEachRow, 'key String, value UInt64') VALUES ('hello', 1), ('world', 2)
```
##### SELECT
```
:) SELECT * FROM url('https://urleng.com/supersecret', JSONEachRow);

┌─key───┬─value─┐
│ hello │     1 │
│ world │     2 │
└───────┴───────┘

2 rows in set. Elapsed: 0.126 sec. 
```

##### URL ENGINE
```
:) CREATE TABLE default.url_engine_distributed
   (
       `key` String,
       `value` UInt64,
   )
   ENGINE = URL('https://urleng.com/supersecret', 'JSONEachRow')
```
```
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
```
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
```
clickhouse-local -q "select count() from url('https://urleng.com/supersecret', JSONEachRow)"
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

(C) 2022 QXIP BV, for more info visit the [qxip metrico](https://metrico.in) repository
