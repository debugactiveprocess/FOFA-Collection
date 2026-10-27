# FOFA API – Postman Collection

A ready-to-use Postman collection for the [FOFA](https://en.fofa.info) cyberspace mapping API. It covers every documented endpoint and ships with **94 requests**, including the hunting queries published in the official [FofaInfo/Awesome-FOFA](https://github.com/FofaInfo/Awesome-FOFA) guides (fuzzy search, threat intelligence discovery, COLDRIVER, Sidewinder APT and more).

## Features

- **Full API coverage**: Search All, Search Next (cursor pagination), Statistic Aggregation, Host Aggregation and Account Information.
- **Write queries in plain text.** A pre-request script encodes the FOFA syntax to URL-safe Base64 (`qbase64`), so you never encode it by hand.
- **Saved hunting queries**: every query from the Awesome-FOFA guides, organized step by step as each guide builds it.
- **Automatic tests**: each response is checked for HTTP 200 and `error: false`, and the Postman console logs the query, total results and returned rows.
- **Cursor handling**: *Search Next* stores the `next` cursor automatically, so you can send the request repeatedly to page through large result sets.
- **API key kept out of requests**: the key lives in an environment variable and is added by collection-level auth.

## Getting started

1. **Import** the collection (`FOFA API`) into Postman, or fork it from the workspace.
2. **Create an environment** named `FOFA` with one variable:

   | Variable   | Type   | Value               |
   |------------|--------|---------------------|
   | `fofa_key` | secret | your FOFA API key   |

3. **Select the `FOFA` environment** in the top-right corner of Postman.
4. **Run `API Reference › Account Information`** to confirm the key works and check your membership level and remaining quota. This call does not consume queries.

> Never commit your API key. Keep it only in your local or private Postman environment.

## How queries work

### Generic requests (API Reference folder)
Set the plain-text FOFA query in the collection variable `query`, for example:

```
domain="example.com" && port="443"
```

The collection pre-request script turns it into `{{qbase64}}` automatically.

### Saved hunting requests
Each saved request contains its query in its own pre-request script:

```js
const q = String.raw`jarm="3fd3fd0003fd3fd21c3fd3fd3fd3fd703dc1bf20eb9604decefea997eabff7" && header="Server: nginx"`;
pm.variables.set('qbase64', encodeURIComponent(CryptoJS.enc.Base64.stringify(CryptoJS.enc.Utf8.parse(q))));
```

To change the query, edit the `const q` line. `String.raw` keeps backslashes and escaped quotes exactly as FOFA expects them (e.g. `banner="\x15\x03\x03\x00\x02\x022"`).

Hunting requests use `full=true`, because most IOCs in the guides date from 2023–2024. That is outside FOFA's default one-year window.

## Collection variables

| Variable       | Default                                             | Purpose |
|----------------|-----------------------------------------------------|---------|
| `baseUrl`      | `https://en.fofa.info/api/v1`                       | API base URL |
| `query`        | `title="bing"`                                      | Plain-text query for the generic requests |
| `qbase64`      | *(generated)*                                       | Base64 query, filled in by the scripts |
| `fields`       | `host,ip,port,protocol,country,title,server`        | Default returned fields |
| `intel_fields` | `host,ip,port,protocol,domain,title,server,country,asn,org,jarm,cert.subject.cn,cert.issuer.cn,cert.not_before,cert.not_after,link` | Fields for threat-hunting requests (all free tier) |
| `size`         | `100`                                               | Results per page (max 10,000) |
| `page`         | `1`                                                 | Page number for Search All |
| `full`         | `false`                                             | `true` searches all history |
| `next_id`      | *(empty)*                                           | Cursor for Search Next (managed automatically) |
| `stats_fields` | `title,country,port,protocol,server`                | Fields for Statistic Aggregation |
| `host`         | `78.48.50.249`                                      | Target for Host Aggregation |

## Collection structure

### 1. API Reference
| Request | Endpoint |
|---------|----------|
| Search All | `GET /search/all` |
| Search Next (continuous pagination) | `GET /search/next` |
| Search All – Static qbase64 example | `GET /search/all` |
| Statistic Aggregation | `GET /search/stats` |
| Host Aggregation / Host Aggregation (detail) | `GET /host/{host}` |
| Account Information | `GET /info/my` |

### 2. Fuzzy Search
Based on *The correct way to use Fuzzy Search in FOFA*. Fuzzy search uses the `*=` operator. Inside the value, `*` matches any number of characters and `?` matches exactly one.

| Example | Query |
|---------|-------|
| Subdomains starting with `test` | `host*="test*.baidu.com"` |
| `test` + one character | `host*="test?.baidu.com"` |
| Five-character subdomains | `host*="?????.baidu.com"` |
| MySQL 5.x versions | `banner="mysql version" && banner*="5.?.*"` |
| All SNMP versions | `protocol*="snmp*"` |
| Cloud provider by partial name | `cloud_name*="Cloud*"` |
| Four-digit ports | `domain="fofa.info" && port*="????"` |

### 3. Threat Intel Discovery Guide
Based on *A Comprehensive Guide to Threat Intelligence Discovery Using FOFA*. The requests show pivoting on headers, banners, ports, JARM and certificates, and how to reduce false positives:

- Header pivot (BLNWX redirect template)
- Cobalt Strike team servers, showing that a matching JARM alone does not prove a C2
- Router `icon_hash` followed by a per-IP SSH banner check
- A generic 404 banner, as an example of what too many results looks like
- Path-collision candidates for the Ducktail IIS C2
- Verification query for OceanLotus (APT32)

### 4. APT – COLDRIVER
Based on *Practical FOFA Asset Discovery: COLDRIVER*. The steps are numbered and run in order:

1. Look up the seed C2 `45.133.216.15:3000`
2. Search the seed C2
3. Match the TLS banner on port 3000
4. Add the default certificate organization (`Internet Widgits Pty Ltd`)
5. Pivot on certificate validity (`2023-06-23 15:59 UTC`)
6. Final rule (intersection of steps 4 and 5): `banner="\x15\x03\x03\x00\x02\x022" && port="3000" && cert="Internet Widgits Pty Ltd" && cert="2023-06-23 15:59 UTC"`
7–8. Look up the two suspicious hosts

### 5. APT – Sidewinder
Based on *Practical FOFA Asset Expansion: Sidewinder APT*.

- Seed IOC lookups: `neger.site`, `semain.tech`, `aliit.org`, `direct888.net`, `fia-gov.net`
- The shared JARM on its own (too broad: about 247k results)
- **Rule 1**: JARM + 404 nginx headers + `Content-Length` 183/535
- **Rule 2**: the `aliit.org` variant (`nginx/1.23.2`, `Content-Length: 555`)
- **Rule 3**: second JARM + `Content-Length: 228`
- A check that the rules still include the seed IOCs, all rules combined, and statistics by country/ASN

### 6. Other Guides
| Folder | Source guide |
|--------|--------------|
| ObserverStealer | Analysis and Tracing of the ObserverStealer |
| APT Bitter | Conducting APT Bitter Tracking Operation Using FOFA |
| APT-C-23 Android Malware | Practical FOFA Asset Expansion: APT-C-23 Android Malware |
| Silver Fox APT | Practical FOFA Asset Expansion: Unmasking the Trail of the 'Silver Fox' APT |
| Ducktail | Practical FOFA Asset Expansion: Ducktail |
| DeepSeek Impersonation | Practical Guide to Discovering DeepSeek Impersonation Assets Using FOFA |
| Certificate Filtering | How to effectively eliminate interference through certificates |
| Supply Chain | Predators in the Supply Chain – Targeting Small and Medium Suppliers |
| Beginner's Guide | A Beginner's Guide |
| FofaHub Examples | Filter Independent IP, Login Discovery, Open Site Discovery |

> The Silver Fox queries keep their Chinese text on purpose, because they must match the impersonated pages exactly.
>
> The LLM/Ollama, Ferret, Ecuador and open-directory articles show their queries only as images or redact them, so they are not included.

## Rate limits and permissions

| Item | Limit |
|------|-------|
| General request rate | below 2 requests/second |
| Statistic Aggregation | 1 request every 5 seconds |
| Host Aggregation | 1 request/second |
| `size` per page | 10,000 max; 2,000 when fields include `cert`/`banner`; 500 when they include `body` |

Some features depend on your membership level:

- **Fuzzy search (`*=`)**: Personal or higher, or F-points.
- **`header_hash`, `banner_hash`, `banner_fid` fields**: Personal or higher.
- **`product`, `lastupdatetime`, `cname` fields**: Professional or higher.
- **`icon_hash`, `body`, `cert.is_valid`, `product.version` fields**: Business or higher.
- **`fid`, `icon`, `structinfo` fields**: Enterprise.

The default field sets in this collection use only free fields. If a request fails because of your membership level, the test results show FOFA's `errmsg`.

## Responsible use

The threat-hunting queries return **leads, not verdicts**. As the Awesome-FOFA guides recommend, cross-check results on threat-intelligence platforms (VirusTotal, ThreatBook, Qianxin, etc.) before you attribute or act on them. Use this collection only for authorized security research, attack-surface management and defensive work, and follow the [FOFA User Agreement](https://i.nosec.org/fofa_service_en_2024).

## References

- FOFA API documentation: https://en.fofa.info/api
- Awesome-FOFA: https://github.com/FofaInfo/Awesome-FOFA
- GoFOFA (official CLI): https://github.com/FofaInfo/GoFOFA
