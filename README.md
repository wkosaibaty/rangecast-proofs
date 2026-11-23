# Rangecast proofs

This repository is a public record of when Rangecast forecasts existed.

Every forecast Rangecast publishes is a JSON document with a SHA-256 hash. Once an hour, and once more for each whole day, the hashes are combined into a single value, a Merkle root, and that root is committed here. A root can only be reproduced from the exact list of hashes it was built from, so a root committed at a given time shows that every forecast in that list already existed, with exactly that content, at that time.

The repository holds roots only. It holds no forecasts and no prices. It says when a forecast existed and that it has not been changed since. It says nothing about whether a forecast turned out to be right.

## What the files are

All times are UTC.

- `hourly/YYYY/MM/DD/THH.json` covers the forecasts published for one hour. `T14.json` under `2026/11/23` is the hour that starts at 14:00 on 23 November 2026.
- `daily/YYYY/MM/DD.json` covers every forecast of one day. It is committed shortly after the day ends.

An hourly file:

```json
{
  "version": 1,
  "kind": "hourly",
  "period": "2026-11-23T14:00:00Z",
  "leaves": 60,
  "root": "<64 lowercase hex characters>",
  "first_issued_at": "2026-11-23T14:00:00Z",
  "last_issued_at": "2026-11-23T14:00:00Z"
}
```

- `period` is the hour, or for a daily file the date (`"2026-11-22"`).
- `leaves` is how many forecast hashes the root covers.
- `root` is the Merkle root of those hashes, in order.
- `first_issued_at` and `last_issued_at` are the earliest and latest `issued_at` among those forecasts.

A daily file has the same fields with `"kind": "daily"`, and one more:

```json
"hours": [
  { "hour": "T00", "leaves": 60, "root": "<hex>" },
  { "hour": "T01", "leaves": 60, "root": "<hex>" }
]
```

The daily list of leaves is each hour's list, one after another, from `T00` to `T23`. `hours` gives the number of leaves each hour contributed and the root of that hour on its own. An hour with no forecasts is left out.

A file is committed once and never edited. If a forecast is corrected, the correction is a new forecast with its own hash in a later file; the earlier one stays as it was. An hour or a day with no forecasts has no file.

## The tree rule

The root is computed as in RFC 6962, section 2.1, with SHA-256.

- A leaf is the 32 bytes of one forecast hash (the 64 hex characters, decoded). The leaves are in the order of Rangecast's publication log, which is the order of the list you are given when you check a forecast.
- `leafHash(leaf) = SHA-256(0x00 || leaf)`
- `nodeHash(left, right) = SHA-256(0x01 || left || right)`
- The root of one leaf is its `leafHash`.
- The root of `n > 1` leaves is `nodeHash(root(leaves[0:k]), root(leaves[k:n]))`, where `k` is the largest power of two strictly less than `n`.
- Roots are written as lowercase hex.

`||` means the bytes are joined. `0x00` and `0x01` are single bytes.

In Python:

```python
import hashlib

def root(leaves: list[bytes]) -> bytes:
    if len(leaves) == 1:
        return hashlib.sha256(b"\x00" + leaves[0]).digest()
    k = 1
    while k * 2 < len(leaves):
        k *= 2
    return hashlib.sha256(b"\x01" + root(leaves[:k]) + root(leaves[k:])).digest()

def root_hex(leaf_hashes: list[str]) -> str:
    return root([bytes.fromhex(h) for h in leaf_hashes]).hex()
```

Two values to test an implementation against. Here leaf `i` is the SHA-256 of the one-character text `0`, `1` or `2`:

| Leaves, in order | Root |
| --- | --- |
| `5feceb66ffc86f38d952786c6d696c79c2dbc239dd4e91b46729d73a27fb57e9` | `13a77175e35eb1d9da91ee14df0d7772cea71289800206e2b45c882ecb06efbf` |
| the leaf above, then `6b86b273ff34fce19d6b804eff5a3f5747ada4eaa22f1d49c01e52ddb7875b4b`, then `d4735e3a265e16eee03f59718b9b5d03019c07d8b6c51f90da3a666eec13ab35` | `8be871f13785b4c81a1700459c76ac2b3ae2caebb7876c376e223c6adff98c47` |

## How to check a forecast

You need two things that are not in this repository: the forecast JSON, and the ordered list of forecast hashes (the leaves) for its hour or its day. Whoever gave you the forecast can give you the list.

1. **Recompute the forecast's hash.** Remove the `hash` field from the forecast JSON. Write what is left as canonical JSON according to RFC 8785 (the JSON Canonicalization Scheme; libraries exist for most languages). Take the SHA-256 of those bytes and write it as lowercase hex. It must equal the `hash` field you removed. If it does not, the document is not the one that was published.
2. **Find the hash among the leaves.** It must appear in the ordered list you were given.
3. **Recompute the root** from the whole list, in the order given, with the rule above.
4. **Compare with the committed file.** Open the file for that hour (`hourly/YYYY/MM/DD/THH.json`, normally the hour of the forecast's `issued_at`) or for that day (`daily/YYYY/MM/DD.json`). Its `root` must equal the root you computed, and its `leaves` must equal the length of your list.
5. **Look at when the file was committed.** The file's history shows one commit, the one that added it, and the time of that commit. That time is the latest moment by which the forecast existed. Compare it with the time the forecast is about.

For a daily file you can also check one hour on its own: the leaves of an hour are a run inside the daily list, starting after the leaves of the hours before it (add up their `leaves` in `hours`), and their root must equal that hour's `root`.

If any step fails, the forecast you hold is not covered by the committed root.

## What a check shows, and what it does not

- It shows that a forecast with exactly this content was among those hashed into a root committed at the time shown.
- It relies on the commit time GitHub shows for the file. If you want a time that does not depend on that, keep your own copy: a clone or a fetch of this repository made on a given day holds every root committed up to that day.
- It covers only the forecasts whose hashes are in the list. It does not show that no other forecast was made in that hour.
- It does not say anything about how accurate a forecast was.
