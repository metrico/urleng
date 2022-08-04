<img src="https://user-images.githubusercontent.com/1423657/182806197-3c20e379-8f1c-4113-bbf4-f3940a7a601a.png" width=800 />


# _"Frugal Metrics"_ 

_Metrics can be lots fun, until you have to share them between a bunch of distributed servers... but fear not!_

<br />

## Distributed timeseries/metrics endpoint with [UrlEng](https://urleng.com) + [ClickHouse](https://clickhouse.com/docs)

Let's build a serverless prometheus metrics store & publisher for ClickHouse using [urleng.com](https://urleng.com)

* no coding, no signups, no logins - batteries included! :battery:	
* zero config distributed table, simply accessible from anywhere
* custom HTTP prometheus /metrics scraping endpoint w/ auto-expiration
* persistent storage courtesy of [deta cloud](https://deta.sh) 

<br/>

## Let's get started!

### Storing metrics in [UrlEng](https://urleng.com)


[UrlEng](https://urleng.com) is a _free serverless pastie_ made for ClickHouse tables.<br>

Let's use it to store a distributed table with a simple schema for our metrics:

| key | help | value | tags |  __expires |
|---  |---          |---    |---   |---        |
| metric name | metric help text | metric value | optional json tags | optional expiration unix ts |

To begin we will keep things basic - performance is not the challenge here but rather flexibiity. A few notes:

* _INSERTs for the same key are considered UPDATEs in URL tables_
* _JSON Tags must be present or empty stringed_
* _Metrics can auto-expire using the `__expires` column and a future unix timestamp_
* **REPLACE `/metrixxx` WITH YOUR OWN UNIQUE STRING OR BE DOOMED**

<br/>

Simple it is! Let's **INSERT** our first little metric:

```sql
INSERT INTO FUNCTION url('https://urleng.com/metrixxx', JSONEachRow, 'key String, help String, value Float64, __expire UInt64, tags String') VALUES ('mygauge', 'my little metric', 100, toUnixTimestamp(now()+300), '{"le":"0.05"}')
```

<br/>

That's easy. Let's take a closer look at our serverless URL table:

```sql
SELECT * FROM url('https://urleng.com/metrixxx', JSONEachRow)

┌─__expires──┬─help──────────────────┬─key──────┬─tags──────────┬─value─┐
│ 1659015366 │ my little metric      │ mygauge  │ {"le":"0.05"} │ 100   │
└────────────┴───────────────────────┴──────────┴───────────────┴───────┘

```

Looking good! Our distributed metrics table is ready. _Get a materialized view up to insert some fun data._

:tooth: ... but the output is still a bit boring and can't be scraped, so its time to change the output format!

<br/>

### FORMAT Prometheus
ClickHouse comes with a built in suport for the Prometheus format - its a little strict, but it works, so let's use it!
```sql
SELECT ifNull(key, 'undefined') as name, ifNull(toFloat64(value),0) as value, help as help, CAST(JSONExtractKeysAndValues(replaceAll(ifNull(tags,''), '\'','\"'), 'String'), 'Map(String, String)') as labels FROM url('https://urleng.com/metrixxx', JSONEachRow) FORMAT Prometheus
```
```
# HELP mygauge my little metric
mygauge{le="0.05"} 100

```

<!--

### FORMAT Template

ClickHouse [templates](https://clickhouse.com/docs/en/interfaces/formats/#format-template) can be used to handle custom defined formats based on flexible template definitions.

Let's use it to output _"prometheus looking"_ metrics from our ClickHouse metric queries:

##### ROW Template
- Create a row format template for prometheus in `/var/lib/clickhouse/format_schemas/row_out.format`
```
# HELP ${0:XML} ${1:XML}
# TYPE ${0:XML} gauge
${0:XML}${3:XML} ${2:XML}
```
##### PAGE Template
- Create a page template to display all data in `/var/lib/clickhouse/format_schemas/prom_out.format`

```
${data}
```

- Does it work? Let's execute a test SELECT using `system.metrics`
```sql
SELECT metric, help, value, '' FROM system.metrics ORDER BY value DESC LIMIT 5 FORMAT Template SETTINGS
format_template_resultset = 'prom_out.format', format_template_row = 'row_out.format', format_template_rows_between_delimiter = '\n'
```

```
# HELP MemoryTracking Total amount of memory (bytes) allocated by the server.
# TYPE MemoryTracking gauge
MemoryTracking 1416436388
# HELP MMappedFileBytes Sum size of mmapped file regions.
# TYPE MMappedFileBytes gauge
MMappedFileBytes 460284336
```

##### All together now!
We're ready. Let's query our URL table metrics using our Prometheus output template:

```sql
SELECT key, help, toFloat64(value), tags 
FROM url('https://urleng.com/metrixxx', JSONEachRow)
ORDER BY value DESC LIMIT 5 FORMAT Template SETTINGS
format_template_resultset = 'prom_out.format', format_template_row = 'row_out.format', format_template_rows_between_delimiter = '\n'
```

```
# HELP mygauge my little metric
# TYPE mygauge gauge
mygauge{type="one"} 100
```

-->

We're almost there - all we need is an endpoint we can scrape and luckily, ClickHouse does that too!

<br/>

### Metrics HTTP Handler

So let's use all the ClickHouse features we can, shall we? <br>
Create a [custom handler](https://clickhouse.com/docs/en/interfaces/http#predefined_http_interface) for our metrics query, ie: ```/etc/clickhouse-server/config.d/metric_export.xml```
```xml
<yandex>
   <format_schema_path>/var/lib/clickhouse/format_schemas/</format_schema_path>
   <custom_urleng>'https://urleng.com/metrixxx</custom_urleng>
   <http_handlers>
      <rule>
        <url>/metrics</url>
        <methods>GET</methods>
        <handler>
            <type>predefined_query_handler</type>
            <query>SELECT ifNull(key, 'undefined') as name, ifNull(toFloat64(value),0) as value, help, CAST(JSONExtractKeysAndValues(replaceAll(ifNull(tags,''), '\'','\"'), 'String'), 'Map(String, String)') as labels FROM url(getSetting('custom_urleng'), JSONEachRow) FORMAT Prometheus</query>
        </handler>
      </rule>
      <defaults/>
   </http_handlers>
</yandex>
```

<br/>

## Scrape it and Shake it!
_Et Voila'!_ Our custom endpoint is ready to be scraped. Let's curl a final test:
```bash
curl 'http://default:password@localhost:8123/metrics'
```
```
# TYPE mygauge gauge
mygauge{le="0.05"} 100

```

:postbox: _Easy and Fun, isn't it?_ Go ahead, update your metrics from any server/service and scrape away! 

<br />

### Bonus Steps

:warning: _This guide is intended as a firestarter - go crazy adding timestamping, output formats and anything else!_

#### URL Engine Table
Going to use your URL engine store daily? Extend the schema and setup a URL Engine table for quicker access:

```sql
CREATE TABLE default.prometheus_exporter
(
 `key` String,
 `value` Float64, 
 `help` String,
 `type` String,
 `tags` String,
 `__expires` UInt64 DEFAULT toUnixTimestamp(now()+300),
)
ENGINE = URL('https://urleng.com/metrixxx', 'JSONEachRow')
 ```
 ```sql
INSERT INTO default.prometheus_exporter VALUES ('mygauge', 100, 'my little metric', 'gauge', '{"le":"0.05"}', toUnixTimestamp(now()+300))
```

----


<!--
#### INSERT w/ Template

Let's parse a fictional query metric
```
some_metric 42 
```

- Create a row format template in `/opt/ch/row_in.format`

```${name:CSV} ${value:CSV}```
  
- Create a page template in `/opt/ch/prom_out.format`

```${data}```

- Execute a query
```
INSERT INTO UserActivity FORMAT Template SETTINGS
format_template_resultset = '/opt/ch/prom_in.format', format_template_row = '/opt/ch/row_in.format'
```

-->

<br/>

## Disclaimers

#### Terms of Use
No warranties of any kind, either express or implied. Data can be removed or lost at any moment of time. Use at your own risk.
#### Security
This service does not provide any security or privacy. Traffic is secured by Cloudflare and data stored on Deta cloud. Use at your own risk.
#### Cookies
This service does not collect, store or use cookies.
#### Limitations
Worker execution time limits might crop large INSERT queries
