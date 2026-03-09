# blackroad-zone-manager

[![CI](https://github.com/BlackRoad-OS/blackroad-zone-manager/actions/workflows/ci.yml/badge.svg)](https://github.com/BlackRoad-OS/blackroad-zone-manager/actions/workflows/ci.yml)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Python 3.9+](https://img.shields.io/badge/python-3.9%2B-blue.svg)](https://www.python.org/downloads/)

DNS zone management library and CLI for the BlackRoad OS platform.  
Manage zones, records, and provider sync — with SQLite persistence, BIND export, and SHA-256 zone checksums.

---

## Table of Contents

- [Overview](#overview)
- [Features](#features)
- [Installation](#installation)
- [Quick Start](#quick-start)
- [CLI Reference](#cli-reference)
  - [create](#create)
  - [add-record](#add-record)
  - [validate](#validate)
  - [export](#export)
  - [import](#import)
  - [sync](#sync)
  - [list](#list)
  - [delete](#delete)
  - [checksum](#checksum)
- [Python API](#python-api)
  - [Zone](#zone-class)
  - [DnsRecord](#dnsrecord-class)
  - [Database](#database)
  - [Record Management](#record-management)
  - [Zone Validation](#zone-validation)
  - [BIND Import / Export](#bind-import--export)
  - [Provider Sync](#provider-sync)
  - [Checksums](#checksums)
- [Supported Record Types](#supported-record-types)
- [DNS Provider Integrations](#dns-provider-integrations)
- [Database Schema](#database-schema)
- [Testing](#testing)
- [Contributing](#contributing)
- [License](#license)

---

## Overview

`blackroad-zone-manager` is a Python library and command-line tool for managing DNS zones on the BlackRoad OS platform.  
It stores zones and records in a local SQLite database, validates DNS record correctness, exports standard BIND zone files, and provides a dry-run sync interface for major DNS providers (Cloudflare, Route 53, DigitalOcean).

---

## Features

- **Zone CRUD** — create, read, update, delete DNS zones with SQLite persistence
- **Record types** — A, AAAA, CNAME, MX, TXT, NS, SOA, SRV, PTR, CAA
- **Validation** — per-record and full-zone validation with descriptive error messages
- **BIND export** — RFC 1035-compliant zone file generation
- **BIND import** — parse existing zone files into the database
- **Provider sync** — dry-run and apply diff against Cloudflare, Route53, DigitalOcean
- **SHA-256 checksums** — deterministic zone fingerprints for change detection
- **Indexed SQLite schema** — `zones` + `records` tables with indexes on zone_id and record type

---

## Installation

```bash
pip install blackroad-zone-manager
```

> **Requirements:** Python 3.9 or later. No external runtime dependencies — only the Python standard library is used.

For development and testing:

```bash
git clone https://github.com/BlackRoad-OS/blackroad-zone-manager.git
cd blackroad-zone-manager
pip install pytest flake8
```

---

## Quick Start

```bash
# Create a zone
python src/main_module.py create example.com --ttl 3600

# Add records
python src/main_module.py add-record example.com A    www  93.184.216.34
python src/main_module.py add-record example.com MX   @    mail.example.com --priority 10
python src/main_module.py add-record example.com TXT  @    "v=spf1 include:blackroad.io ~all"

# Validate
python src/main_module.py validate example.com

# Export as BIND zone file
python src/main_module.py export example.com -o example.com.zone

# Sync with Cloudflare (dry-run)
python src/main_module.py sync example.com --provider cloudflare

# List all zones
python src/main_module.py list
```

---

## CLI Reference

All commands accept a global `--db <path>` option to override the default SQLite database location (`~/.blackroad/zone-manager.db`).

### create

Create a new DNS zone.

```
python src/main_module.py create <name> [--ttl TTL]
```

| Argument | Required | Default | Description |
|----------|----------|---------|-------------|
| `name`   | ✓        | —       | Zone name (e.g. `example.com`) |
| `--ttl`  |          | `3600`  | Default TTL for the zone in seconds |

### add-record

Add a DNS record to an existing zone.

```
python src/main_module.py add-record <zone> <type> <name> <value> [--ttl TTL] [--priority PRI]
```

| Argument      | Required | Default | Description |
|---------------|----------|---------|-------------|
| `zone`        | ✓        | —       | Zone name |
| `type`        | ✓        | —       | Record type (A, AAAA, CNAME, MX, TXT, NS, SOA, SRV, PTR, CAA) |
| `name`        | ✓        | —       | Record label (`@` for apex, `www`, `mail`, …) |
| `value`       | ✓        | —       | Record value |
| `--ttl`       |          | `300`   | TTL in seconds |
| `--priority`  |          | `None`  | Priority (required for MX and SRV records) |

### validate

Validate all records in a zone and report errors.

```
python src/main_module.py validate <name>
```

Exits with code `1` if any errors are found.

### export

Export a zone in BIND/RFC 1035 format.

```
python src/main_module.py export <name> [--output FILE]
```

If `--output` is omitted the zone file is printed to stdout.

### import

Import a BIND zone file into the database.

```
python src/main_module.py import <path>
```

### sync

Diff the zone against a DNS provider and optionally apply changes.

```
python src/main_module.py sync <name> [--provider PROVIDER] [--apply]
```

| Argument      | Default      | Description |
|---------------|--------------|-------------|
| `--provider`  | `cloudflare` | Target provider (`cloudflare`, `route53`, `digitalocean`) |
| `--apply`     | dry-run      | Apply changes; omit for dry-run only |

### list

List all zones stored in the database.

```
python src/main_module.py list
```

### delete

Delete a zone and all its records.

```
python src/main_module.py delete <name>
```

### checksum

Print the SHA-256 fingerprint of a zone's current content.

```
python src/main_module.py checksum <name>
```

---

## Python API

```python
from src.main_module import (
    Zone, DnsRecord, get_db,
    save_zone, load_zone, list_zones, delete_zone,
    add_record, remove_record, validate_record, validate_zone,
    export_bind_format, import_zone_file,
    sync_with_provider, zone_checksum,
)
```

### Zone class

```python
@dataclass
class Zone:
    name: str                  # e.g. "example.com"
    ttl: int = 3600            # default TTL in seconds
    serial: int = ...          # auto-generated (YYYYMMDDnn)
    nameservers: list = [...]  # defaults to blackroad.io nameservers
    records: list = []
    zone_id: Optional[int] = None
```

### DnsRecord class

```python
@dataclass
class DnsRecord:
    record_type: str           # A, AAAA, CNAME, MX, TXT, NS, SOA, SRV, PTR, CAA
    name: str                  # relative label, "@" for apex
    value: str
    ttl: int = 300
    priority: Optional[int] = None  # required for MX / SRV
    record_id: Optional[int] = None
```

### Database

```python
# Open (or create) the database, initialise schema
conn = get_db()                          # default path: ~/.blackroad/zone-manager.db
conn = get_db(Path("/custom/path.db"))   # custom path
```

### Record Management

```python
# Save a new zone; populates zone.zone_id
zone = Zone(name="example.com", ttl=3600)
save_zone(zone, conn)

# Load a zone with all records
zone = load_zone("example.com", conn)   # returns None if not found

# Add a record (validates before inserting)
rec = add_record(zone, "A", "www", "93.184.216.34", ttl=300, conn=conn)
rec = add_record(zone, "MX", "@", "mail.example.com.", priority=10, conn=conn)

# Remove a record by its record_id
remove_record(zone, rec.record_id, conn=conn)

# List all zones
zones = list_zones(conn)  # list of dicts: {name, ttl, serial, created_at}

# Delete a zone (cascades to records)
delete_zone("example.com", conn)  # returns True if found
```

### Zone Validation

```python
# Validate a single record — returns list of error strings
errors = validate_record(rec)

# Validate an entire zone (checks SOA, NS presence, per-record rules, duplicate CNAMEs)
errors = validate_zone(zone)
if errors:
    for e in errors:
        print(e)
```

### BIND Import / Export

```python
# Export to BIND zone file format
bind_text = export_bind_format(zone)
Path("example.com.zone").write_text(bind_text)

# Import an existing zone file
zone = import_zone_file("/path/to/example.com.zone", conn=conn)
```

### Provider Sync

```python
result = sync_with_provider(zone, provider="cloudflare", dry_run=True)
# result keys: provider, zone, dry_run, records_to_create,
#              records_to_update, records_to_delete, errors, total_changes
print(result["total_changes"])

# Apply changes (real API calls in production implementation)
result = sync_with_provider(zone, provider="route53", dry_run=False)
```

Supported providers: `cloudflare`, `route53`, `digitalocean`.

### Checksums

```python
# SHA-256 fingerprint of the full zone content (deterministic)
checksum = zone_checksum(zone)  # 64-character hex string
```

---

## Supported Record Types

| Type  | Validation                          | Priority required |
|-------|-------------------------------------|-------------------|
| A     | IPv4 address                        | No                |
| AAAA  | IPv6 address (hex colon notation)   | No                |
| CNAME | Hostname                            | No                |
| MX    | Hostname                            | **Yes**           |
| TXT   | Any string                          | No                |
| NS    | Hostname                            | No                |
| SOA   | Any string                          | No                |
| SRV   | Any string                          | **Yes**           |
| PTR   | Hostname                            | No                |
| CAA   | Any string                          | No                |

---

## DNS Provider Integrations

The `sync_with_provider` function provides a diff/apply interface for three providers.  
The current implementation computes the changeset (dry-run) and is designed to be extended with provider REST API calls.

| Provider     | Identifier     | Auth environment variable (planned) |
|--------------|----------------|--------------------------------------|
| Cloudflare   | `cloudflare`   | `CLOUDFLARE_API_TOKEN`               |
| AWS Route 53 | `route53`      | `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` |
| DigitalOcean | `digitalocean` | `DIGITALOCEAN_TOKEN`                 |

---

## Database Schema

The SQLite database is stored at `~/.blackroad/zone-manager.db` by default.

```sql
CREATE TABLE zones (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    name        TEXT    NOT NULL UNIQUE,
    ttl         INTEGER NOT NULL DEFAULT 3600,
    serial      INTEGER NOT NULL,
    nameservers TEXT    NOT NULL DEFAULT '[]',
    created_at  TEXT    NOT NULL DEFAULT (datetime('now')),
    updated_at  TEXT    NOT NULL DEFAULT (datetime('now'))
);

CREATE TABLE records (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    zone_id     INTEGER NOT NULL REFERENCES zones(id) ON DELETE CASCADE,
    record_type TEXT    NOT NULL,
    name        TEXT    NOT NULL,
    value       TEXT    NOT NULL,
    ttl         INTEGER NOT NULL DEFAULT 300,
    priority    INTEGER,
    created_at  TEXT    NOT NULL DEFAULT (datetime('now'))
);

CREATE INDEX idx_records_zone ON records(zone_id);
CREATE INDEX idx_records_type ON records(record_type);
```

---

## Testing

### Unit tests

```bash
pip install pytest
python -m pytest tests/ -v
```

### Linting

```bash
pip install flake8
flake8 src/ --max-line-length=120 --ignore=E501,W503
```

### End-to-end (E2E) test

The following sequence exercises the full CLI pipeline:

```bash
DB=/tmp/e2e-test.db

# 1. Create zone
python src/main_module.py --db $DB create e2e.example.com --ttl 3600

# 2. Add records
python src/main_module.py --db $DB add-record e2e.example.com A    www  203.0.113.1
python src/main_module.py --db $DB add-record e2e.example.com MX   @    mail.e2e.example.com --priority 10
python src/main_module.py --db $DB add-record e2e.example.com TXT  @    "v=spf1 ~all"

# 3. List
python src/main_module.py --db $DB list

# 4. Validate (expected: reports missing SOA/NS)
python src/main_module.py --db $DB validate e2e.example.com || true

# 5. Export
python src/main_module.py --db $DB export e2e.example.com -o /tmp/e2e.zone
cat /tmp/e2e.zone

# 6. Checksum
python src/main_module.py --db $DB checksum e2e.example.com

# 7. Sync dry-run
python src/main_module.py --db $DB sync e2e.example.com --provider cloudflare

# 8. Delete
python src/main_module.py --db $DB delete e2e.example.com
python src/main_module.py --db $DB list

rm -f $DB /tmp/e2e.zone
```

---

## Contributing

1. Fork the repository and create a feature branch.
2. Write or update tests in `tests/test_module.py`.
3. Ensure linting passes: `flake8 src/ --max-line-length=120 --ignore=E501,W503`
4. Ensure all tests pass: `python -m pytest tests/ -v`
5. Open a pull request — CI will run automatically.

---

## License

[MIT](LICENSE) © BlackRoad OS

