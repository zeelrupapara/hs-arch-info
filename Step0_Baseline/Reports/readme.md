## Service Directories

Each folder inside `logs/` corresponds to a backend service in the system.

| Service Folder | Description | Top-5 Errors File |
|----------------|-------------|-------------------|
| **vfxcore** | Core engine operations, order processing, validation logic. | [errors_top5_YYYYMMDD.md](logs/vfxcore/errors_top5_YYYYMMDD.md) |
| **vfxdata** | Storage layer, caching, DB sync, data pipeline issues. | [errors_top5_YYYYMMDD.md](logs/vfxdata/errors_top5_YYYYMMDD.md) |
| **vfxmarket** | Market feed ingestion, price updates, symbol data failures. | [errors_top5_YYYYMMDD.md](logs/vfxmarket/errors_top5_YYYYMMDD.md) |
| **vfxserver** | Main API server, routing, gateway, client-facing errors. | [errors_top5_YYYYMMDD.md](logs/vfxserver/errors_top5_YYYYMMDD.md) |

Every folder may contain:
- Raw log files  
- Extracted error summaries  
- `errors_top5_YYYYMMDD.md` (latest top five errors for that service)

---

## errors_top5_YYYYMMDD.md

Each service folder maintains a Markdown file following this naming format: `errors_top5_YYYYMMDD.md`

### Structure

The file contains the top 5 most frequent errors from the last 72 hours, one per line in the following format:

```
timestamp | service | message | correlation_id
```

**Fields:**
- **timestamp**: Error timestamp in format `YYYY-MM-DD HH:MM:SS UTC`
- **service**: Service name (vfxcore, vfxdata, vfxmarket, vfxserver)
- **message**: Error message text
- **correlation_id**: Correlation ID if available, otherwise `N/A`

**Example:**
```
2025-11-13 11:25:59 UTC | vfxserver | Error from provider dailyfx: XML syntax error on line 9: element <P> closed by </BODY> | N/A
2025-11-13 11:18:34 UTC | vfxserver | expired or invalid token | N/A
2025-11-13 11:18:35 UTC | vfxserver | session is already used | N/A
```

### Generation

The error reports are generated using the `generate_error_reports.py` script:

```bash
python3 generate_error_reports.py
```

The script:
- Processes the latest 5 log files per service (or all available if less than 5)
- Extracts errors from the last 72 hours
- Groups errors by message and identifies the top 5 most frequent
- Generates `errors_top5_YYYYMMDD.md` files for each service

**Purpose:** Provides raw error evidence for mapping against A1/A2 taxonomy.

