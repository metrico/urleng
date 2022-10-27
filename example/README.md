# URLENG
## Local Server Examples

The included example illustrates a minimalistic ClickHouse [URL Engine backend](https://content.clickhouse.tech/docs/en/engines/table-engines/special/url/) w/ [Deta](https://deta.sh) and [Memory](mem.js) storage backend.

Note: this repository does _not_ include the cloudflare workers used by the [urleng.com](https://urleng.com) service.

## Usage
```bash
npm install
```

### Memory only
Start a URLENG server w/o persistence
```bash
node urleng-mem.js
```
### Cloud Storage
Start a URLENG server w/ [Deta](https://deta.sh) persistence. _Requires a valid 'TOKEN'_
```bash
DETA_TOKEN=XXXYYYZZZ node urleng-deta.js
```