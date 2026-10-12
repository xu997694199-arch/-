# Preserving measurement time when dashboards combine snapshots

A status page can show the right number and still tell the wrong story. If an exporter copies an old measurement into a newly generated file, then labels that measurement with the file's creation time, the value suddenly looks current. Nothing about the underlying system has been measured again.

I encountered this in a local status exporter that combined two independent sources. One supplied task progress; the other supplied a periodically collected usage measurement. Refreshing task progress advanced the overall snapshot timestamp. The renderer then reused that timestamp for the usage row. The percentage was unchanged, but its apparent age had been reset.

The fix was small: preserve the usage source's collection time. The more interesting part was deciding what to do with older records and explicitly unknown timestamps. Those cases deserve separate treatment.

## Three times can describe one row

Consider an illustrative sequence:

| Event | Time |
| --- | --- |
| A collector observes usage | 09:00 UTC |
| Another collector refreshes task progress | 09:45 UTC |
| An exporter writes the combined dashboard | 10:00 UTC |

The usage row is an hour old at export time. A footer saying “page generated at 10:00 UTC” is accurate. A usage label saying “measured at 10:00 UTC” is not.

I would keep three concepts distinct:

- **Observation time:** when the source was measured.
- **Snapshot time:** when a collector captured or assembled a record.
- **Export time:** when a view was rendered or copied.

In the example below, the usage collector records its observation time as `usage_captured_utc`. A production source may have a separate provider timestamp and fetch timestamp; if so, preserve both and label them precisely. Fetching a cached provider response does not make the underlying observation new.

This is a provenance problem before it is a formatting problem. Once a pipeline discards the source time, a renderer cannot reconstruct it from the output file's modification time.

## Missing and null need different meanings

The existing exporter had historical records with only `captured_utc`. Newer records could carry a separate `usage_captured_utc`.

For the illustrative migration below, I use this explicit contract:

| Input | Interpretation |
| --- | --- |
| `usage_captured_utc` is present with a timestamp | Use that source timestamp |
| The key is absent | Use the legacy capture time, and label the fallback |
| The key is present with `null` | Observation time is unknown |
| The key contains malformed data | Timestamp is invalid; do not substitute a fresher value |

That distinction rules out a tempting shortcut:

`record.get("usage_captured_utc") or record.get("captured_utc")`

The expression treats an explicit null like an absent key. It would also hide an empty string. In this contract, null says “unknown”; it does not invite a replacement.

The legacy fallback itself is conditional. It is defensible only if the historical collector's capture time actually represented the usage collection time closely enough for the application. If that cannot be established, old records should remain unknown. A compatibility default is not evidence of freshness.

## A small reproducible example

The following example uses Python's standard library. All times and percentages are synthetic. It intentionally models only timestamp selection and a display age; it does not contact a service.

The fixed reference time makes the checks repeatable. In an application, the renderer would supply its current UTC time instead.

```python
from copy import deepcopy
from datetime import datetime, timezone

NOW = datetime(2026, 1, 1, 10, 0, tzinfo=timezone.utc)


def source_time(record):
    if "usage_captured_utc" in record:
        return record["usage_captured_utc"], "usage observation"
    return record.get("captured_utc"), "legacy snapshot"


def usage_view(record, now):
    if now.tzinfo is None or now.utcoffset() is None:
        raise ValueError("now must have an explicit timezone")

    timestamp, origin = source_time(record)
    view = {
        "used_percent": record.get("used_percent"),
        "observed_at": timestamp,
        "time_origin": origin,
        "age_seconds": None,
        "time_state": "unknown",
    }
    if timestamp is None:
        return view

    try:
        observed = datetime.fromisoformat(timestamp)
        if observed.tzinfo is None or observed.utcoffset() is None:
            raise ValueError("observation must have an explicit timezone")
        age = (now - observed).total_seconds()
    except (TypeError, ValueError):
        view["time_state"] = "invalid"
        return view

    if age < 0:
        view["time_state"] = "future timestamp"
        return view

    view["age_seconds"] = age
    view["time_state"] = "known"
    return view


sample = {
    "used_percent": 42,
    "captured_utc": "2026-01-01T09:45:00+00:00",
    "usage_captured_utc": "2026-01-01T09:00:00+00:00",
}

# Refreshing unrelated progress must not refresh the usage observation.
before = usage_view(sample, NOW)
later = deepcopy(sample)
later["captured_utc"] = "2026-01-01T10:00:00+00:00"
assert usage_view(later, NOW) == before
assert before["age_seconds"] == 3600
print("PASS unrelated refresh preserves the one-hour observation age")

# The historical fallback must be visible to downstream presentation.
legacy = {"used_percent": 42, "captured_utc": sample["usage_captured_utc"]}
legacy_view = usage_view(legacy, NOW)
assert legacy_view["time_origin"] == "legacy snapshot"
assert legacy_view["age_seconds"] == 3600
print("PASS missing key uses the explicitly labelled legacy fallback")

# Explicit uncertainty must survive the migration.
unknown = dict(sample, usage_captured_utc=None, used_percent=None)
unknown_view = usage_view(unknown, NOW)
assert unknown_view["time_state"] == "unknown"
assert unknown_view["age_seconds"] is None
assert unknown_view["used_percent"] is None
print("PASS explicit null preserves unknown time and unknown value")

for bad_time in ("", "not-a-date", 123, "2026-01-01T09:00:00"):
    invalid = usage_view(dict(sample, usage_captured_utc=bad_time), NOW)
    assert invalid["time_state"] == "invalid"
    assert invalid["age_seconds"] is None
print("PASS malformed and timezone-free timestamps stay invalid")

future = usage_view(
    dict(sample, usage_captured_utc="2026-01-01T10:01:00+00:00"), NOW
)
assert future["time_state"] == "future timestamp"
assert future["age_seconds"] is None
print("PASS a future observation is flagged instead of called fresh")

# Equal instants with different UTC offsets must have equal ages.
offset = usage_view(
    dict(sample, usage_captured_utc="2026-01-01T11:00:00+02:00"), NOW
)
assert offset["age_seconds"] == before["age_seconds"]
print("PASS equivalent timezone offsets preserve observation age")
```

The first check expresses the bug as an invariant: changing an unrelated snapshot time must leave the usage view unchanged when the rendering reference time is fixed. That is stronger than checking whether a particular field was added to the output.

The explicit-null check also protects the number itself. An unavailable percentage remains `None`; it does not become zero. This function passes other measurement values through unchanged. Range checks, units, and provider-specific validation belong at the input boundary and are outside this small example.

## Make uncertainty visible in the UI

For a normal row, show the value and the source age together: “42%, observed 60 minutes ago.” A separate footer can show when the page was rendered.

For a legacy record, retain a visible qualifier such as “time from legacy snapshot.” For unknown, invalid, or future times, show the corresponding state instead of a reassuring “just now.” Do not clamp a negative age to zero: it can conceal clock skew or an upstream timestamp error.

Whether a known age is acceptable depends on the decision. A five-minute observation might be fine for a daily report and unsuitable for a rapidly changing capacity limit. Keep the stale threshold configurable and separate from the timestamp parser. The example deliberately does not decide when data is fresh.

If observation clocks are not trustworthy, a consumer should also retain its own receipt time. Receipt time can answer how long the consumer has held the response. It cannot prove when the source value was measured.

## Rewriting a file is not another measurement

Copying a snapshot to another folder, regenerating a dashboard, or reformatting its JSON should preserve the source timestamp. File metadata can verify that an export happened; it cannot establish that every exported value was observed again.

For the local exporter, I kept the change narrow: the usage row read its own timestamp, the missing-key fallback was retained while explicit null stayed unknown, and updating task progress no longer changed the usage date. I did not need to replace the collector or add a database to fix that behavior. The labelled fallback and invalid-time states in this article are additional safeguards demonstrated by the example.

If I were reviewing another combined status page, the first question would be simple: can an unrelated refresh make any measurement look younger? I would test that before spending time on the dashboard's colors. A polished number without its real age is still an unreliable input to a decision.

---

Reproduction: the single Python block above is self-contained. This article's example was run under Python 3.13; the recorded checks cover timestamp propagation and the stated edge cases, not the behavior of an external provider.

Reference: [Python datetime documentation](https://docs.python.org/3.13/library/datetime.html).

