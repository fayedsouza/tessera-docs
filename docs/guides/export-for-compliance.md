# Export for compliance

Produce a signed archive of events that an auditor can verify independently.

Requires a token with the `exports:create` scope.

## When to export rather than query

Export when you need a complete, fixed, verifiable set of records:

- An auditor has asked for everything touching a system over a period.
- You are retaining records beyond your workspace retention horizon.
- The result set is large — above roughly ten thousand events, paging the query API is
  slower and less reliable than one export.

An export is a point-in-time snapshot. Because the log is append-only, re-running the
same export later returns a superset, never a different version of the same records.

## Create an export

An export is defined by the same filters as a query:

```bash
curl -X POST https://api.tessera.example/v1/exports \
  -H "Authorization: Bearer $TESSERA_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "format": "jsonl",
    "filters": {
      "resource_type": "document",
      "occurred_after": "2026-01-01T00:00:00Z",
      "occurred_before": "2026-07-01T00:00:00Z"
    },
    "note": "H1 2026 document access - FCA request ref 4471"
  }'
```

```json
{
  "id": "exp_01JB9T4M2Z",
  "status": "pending",
  "created_at": "2026-09-27T11:02:14Z",
  "note": "H1 2026 document access - FCA request ref 4471"
}
```

Use `note` for the reason and any external reference. It is stored with the export and
appears in the manifest, which saves reconstructing months later why an archive exists.

Creating an export is itself recorded as an event in your log.

### Formats

| Format | Use for |
|---|---|
| `jsonl` | One JSON object per line. The default, and the only format preserving the full `context` object |
| `csv` | Spreadsheet review. Nested fields are flattened with dots; `context` is serialised into a single column |
| `parquet` | Loading into a warehouse |

Prefer `jsonl` for anything going to an auditor. CSV flattening is lossy where `context`
holds nested structures, and you will not notice until someone asks about a field that is
no longer there.

## Poll for completion

Exports run asynchronously. Small ones finish in seconds; a multi-million-event export
can take the better part of an hour.

```bash
curl https://api.tessera.example/v1/exports/exp_01JB9T4M2Z \
  -H "Authorization: Bearer $TESSERA_TOKEN"
```

| Status | Meaning |
|---|---|
| `pending` | Queued |
| `running` | In progress. `progress_pct` is populated |
| `complete` | Ready. `download_url` and `manifest` are populated |
| `failed` | See `error`. Retry with a narrower filter |
| `expired` | Older than 7 days; the archive has been deleted |

Poll no more than once every 10 seconds. Faster polling counts against your rate budget
and will not make the export finish sooner.

A completed export:

```json
{
  "id": "exp_01JB9T4M2Z",
  "status": "complete",
  "event_count": 418223,
  "download_url": "https://exports.tessera.example/exp_01JB9T4M2Z.jsonl.gz?...",
  "expires_at": "2026-10-04T11:09:52Z",
  "manifest": {
    "sha256": "9f2c4a...c81e",
    "signature": "MEUCIQD...",
    "signing_key_id": "key_2026_q3",
    "sequence_range": { "first": 12004, "last": 430226 }
  }
}
```

## Download and verify

The `download_url` is pre-signed and valid for one hour. The archive itself is deleted
after seven days — download it and store it somewhere you control.

```bash
curl -o h1-2026-access.jsonl.gz "$DOWNLOAD_URL"
```

Verify the checksum:

```bash
sha256sum h1-2026-access.jsonl.gz
# compare against manifest.sha256
```

Verify the signature against Tessera's published key:

```bash
curl -o tessera-key_2026_q3.pem \
  https://api.tessera.example/v1/signing-keys/key_2026_q3

openssl dgst -sha256 -verify tessera-key_2026_q3.pem \
  -signature signature.bin h1-2026-access.jsonl.gz
```

!!! tip "Record the verification, not just the archive"
    Auditors generally want evidence that *you* checked the signature, not only that one
    exists. Store the verification output alongside the archive with the date it was
    performed.

The manifest's `sequence_range` lets a reader confirm completeness: every sequence number
between `first` and `last` that matches the filter is present in the file. A gap means
the events in between did not match the filter, not that anything was removed.

## Schedule recurring exports

To retain records beyond your retention horizon, schedule an export rather than
remembering to run one:

```bash
curl -X POST https://api.tessera.example/v1/exports/schedules \
  -H "Authorization: Bearer $TESSERA_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "cadence": "monthly",
    "format": "jsonl",
    "filters": {},
    "destination": {
      "type": "s3",
      "bucket": "acme-audit-archive",
      "prefix": "tessera/",
      "role_arn": "arn:aws:iam::123456789012:role/TesseraExport"
    }
  }'
```

Scheduled exports deliver straight to your bucket, so nothing depends on someone
downloading within the seven-day window. Set this up at integration time —
[retention expiry is deletion, not archival](../concepts/events-and-retention.md#expiry-is-deletion-not-archival).

## Related

- [Query the audit log](query-the-audit-log.md)
- [Events and retention](../concepts/events-and-retention.md)
