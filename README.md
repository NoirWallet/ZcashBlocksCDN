# Zcash Compact Block CDN

A public, read-only HTTP service for downloading **historical Zcash compact blocks**. Wallets and synchronization services can use the CDN to accelerate block scanning and reduce the number of historical `GetBlockRange` requests sent to lightwalletd.

**No account, API token, or R2 credentials are required.**

**Mainnet base URL:**

```text
https://zcashblocks.zknoir.com/compact-blocks/v1
```

> **Scope:** The CDN serves compact blocks only. Chain-tip information, tree states, subtree roots, transparent address history, full transactions, and other RPC data must still be obtained from lightwalletd. The CDN is not a complete lightwalletd replacement.

## 1. Background: Why Build a Compact Block CDN?

A Zcash light wallet must scan compact blocks from its wallet birthday to discover shielded transactions. New wallet setup, seed-phrase recovery, and rescanning can require downloading a large amount of historical data. Traditionally, clients obtain these blocks from **lightwalletd** using `GetBlockRange`.

That model is appropriate for live RPC data, but historical compact blocks are **read-heavy and shared by every client**: many wallets repeatedly request the same immutable blocks. Serving all of those requests through lightwalletd increases backend traffic, outbound bandwidth, and sensitivity to RPC concurrency or network latency. For browser wallets, downloading static files over ordinary HTTPS can also be simpler than maintaining a gRPC-Web data path for bulk historical data.

**Our approach is to separate historical data distribution from live blockchain RPC:** publish compact blocks as verifiable, versioned HTTP objects and deliver them through a CDN. This is intended to reduce repeated load on lightwalletd, make parallel and cached downloads practical, and improve the scalability of wallet synchronization. It does **not** replace lightwalletd, independent chain validation, or wallet scanning.

## 2. Architecture and Why Cloudflare R2

### Architecture overview

The service has two logical paths: **historical data publication** and **public data delivery**.

```
flowchart TB
    subgraph Publishing["Historical data publication"]
        Source["Zcash node / lightwalletd"] --> Exporter["Export and validate CompactBlocks"]
        Exporter --> Chunker["Build protobuf chunks (currently 500-block alignment)"]
        Chunker --> Upload["Upload immutable chunks and indexes"]
        Upload --> Manifest["Publish manifest.json last"]
    end
    Upload --> R2[("Cloudflare R2 object storage")]
    Manifest --> R2
    subgraph Delivery["Public data delivery"]
        Client["Wallet / synchronization client"] -->|"HTTPS GET"| Edge["Cloudflare edge cache"]
        Edge -->|"Cache miss"| R2
        Client -->|"Chain tip, tree state, other RPCs, fallback"| LWD["lightwalletd"]
    end
```

The publishing pipeline is shown at a logical level; exporters and edge routing can be implemented independently of the public file format.

1. **Build:** Retrieve historical `CompactBlock` messages equivalent to `GetBlockRange(poolTypes=[])`, group them into height-aligned protobuf chunks, and record integrity metadata.
2. **Index:** Create JSON index files listing the chunks; create a manifest describing the published block range and referencing those indexes. Index and chunk filenames include content hashes.
3. **Publish:** Upload the new chunks and indexes to R2 first, then update `manifest.json` **last**. Existing hash-named objects are treated as immutable so that cached clients can continue using them.
4. **Deliver:** Clients download the manifest, indexes, and selected chunks over HTTPS. Cloudflare's edge cache can serve repeated requests, with R2 as the backing object store on a cache miss.
5. **Fallback:** Requests for recent/unpublished blocks, other RPC data, or failed CDN downloads continue through lightwalletd.

### Why Cloudflare R2?

| Consideration | Rationale |
| --- | --- |
| **Well matched to immutable history** | Compact-block chunks are static objects, so an object store is a better fit for public distribution than serving every historical request from an RPC process. |
| **Predictable outbound bandwidth economics** | Cloudflare R2 does not charge internet egress fees. Storage, read/write operations, and any separately billed Cloudflare services **can still incur charges**. |
| **Cloudflare edge delivery** | A custom domain plus appropriately configured Cache Rules allows repeat requests to be served from Cloudflare edge locations rather than fetching every object from R2. |
| **Simple publishing and access** | R2 provides an S3-compatible API for authenticated uploads, while users can retrieve public files over HTTPS without R2 credentials. |
| **Cache-friendly versioning** | Hash-named index/chunk URLs are immutable; only the manifest needs frequent refreshes, which simplifies safe long-lived caching. |

**Cache policy matters:** R2 is the origin store, **not automatically a worldwide cache for every file type**. Configure Cloudflare caching explicitly for the JSON and `.pb` endpoints, especially `.pb` files. Use a short TTL for the mutable `manifest.json` and long TTLs for immutable hash-named indexes and chunks. CDN caching can temporarily serve an older manifest even though R2 itself offers strong read-after-write consistency; clients should rely on the manifest they actually received and fall back to lightwalletd when necessary.

A Cloudflare Worker may be used for path routing, response headers, or customized caching, but **the public protocol does not require a Worker or expose R2 credentials to clients**. For browser integrations, ensure the public endpoint's CORS policy permits the required reads.

## 3. Quick Start

Fetch the mainnet manifest to discover the currently published block range and index files:

```bash
BASE_URL="https://zcashblocks.zknoir.com/compact-blocks/v1"
DATASET_URL="$BASE_URL/main/shielded-default"

curl -fsSL --compressed "$DATASET_URL/manifest.json" \
  | jq '{format_version, network, first_height, published_through, chunk_size, indexes}'
```

Requires `curl` and `jq`. The response is JSON.

The CDN covers heights from **`first_height` through `published_through`, inclusive**. Equivalently, the covered half-open interval is:

```text
[first_height, published_through + 1)
```

Always read the current manifest. Coverage can change; do not hardcode a block height, index URL, or chunk filename.

## 4. URL Structure

```text
<base_url>/<network>/shielded-default/manifest.json
<base_url>/<network>/shielded-default/indexes/<start>-<end_exclusive>-<sha256>.json
<base_url>/<network>/shielded-default/chunks/<start>-<end_exclusive>-<sha256>.pb
```

- **Network:** `main` for the published mainnet dataset. If a `test` dataset is offered in the future, first check that its manifest exists.
- **Profile:** `shielded-default`, corresponding to `GetBlockRange(poolTypes=[])`.
- **Paths:** The `path` fields in the manifest and indexes are relative to `<base_url>/<network>/shielded-default/`. Always use the paths supplied in the metadata; do not construct filenames from block heights.

## 5. Data Organization

```text
manifest.json
    └── indexes[]
          └── index.json
                └── chunks[]
                      └── chunk.pb (multiple CompactBlock messages)
```

### Manifest (`manifest.json`)

| Field | Description |
| --- | --- |
| `format_version` | Metadata format version; currently `1` |
| `network`, `genesis_hash` | Chain identity; genesis hash is a hex string |
| `profile`, `pool_types` | Currently `shielded-default` and `[]` |
| `first_height`, `published_through` | Published coverage, both inclusive |
| `chunk_size` | Chunk alignment, currently 500 blocks |
| `index_span` | Index grouping window; not the client request size |
| `publication_lag` | Publication policy, **not** a finality guarantee |
| `generated_at_unix_seconds` | Manifest creation time (Unix seconds) |
| `indexes[]` | Index references: `start_height`, `end_height_exclusive`, `path`, `sha256` |

### Index (`indexes/*.json`)

Each index contains `format_version`, `network`, `profile`, `start_height`, `end_height_exclusive`, and `chunks[]`.

Each chunk entry provides:

| Field | Description |
| --- | --- |
| `start_height`, `end_height_exclusive` | Chunk coverage: `[start_height, end_height_exclusive)` |
| `path` | Dataset-relative path to the `.pb` file |
| `sha256` | SHA-256 of the **HTTP-decompressed** `.pb` bytes |
| `uncompressed_size` | Size in bytes after HTTP decompression |
| `first_block_hash`, `last_block_hash` | Hashes of the first and last blocks, in hex |

The manifest's `indexes[].sha256` similarly refers to the HTTP-decompressed index JSON bytes.

### Chunk (`chunks/*.pb`)

A chunk is **not a gRPC-Web response**. It is a Protocol Buffers envelope containing multiple standard lightwalletd `CompactBlock` messages.

```proto
syntax = "proto3";
import "compact_formats.proto";

message CompactBlockChunk {
  uint32 format_version = 1;
  string network = 2;
  bytes genesis_hash = 3;
  string profile = 4;
  uint64 start_height = 5;
  uint64 end_height_exclusive = 6;
  bytes first_block_hash = 7;
  bytes last_block_hash = 8;
  repeated .cash.z.wallet.sdk.rpc.CompactBlock blocks = 9;
}
```

`CompactBlock` is defined in the official Zcash [compact_formats.proto](https://github.com/zcash/lightwallet-protocol/blob/main/walletrpc/compact_formats.proto). Add that file to your Protobuf compiler's import path when compiling the envelope definition.

## 6. Downloading a Block Range

For a requested half-open range `[start, end)`:

1. Download the current manifest and intersect the requested range with `[first_height, published_through + 1)`.
2. Select every index satisfying `index.start_height < end && index.end_height_exclusive > start`.
3. Download the matching indexes and verify each one's SHA-256 against the manifest.
4. Within those indexes, select chunks using the same interval-overlap rule.
5. Download **each complete chunk**, verify its decompressed byte count and SHA-256, and decode its Protobuf envelope.
6. Extract the requested block heights, validate continuity, and return blocks in ascending order.
7. Fetch any heights outside CDN coverage, or affected by missing files, timeouts, or validation failures, from **lightwalletd**.

Even if you need only one block from a chunk, you must download that entire `.pb` file. The current 500-block alignment is a storage detail, not a restriction on client request ranges.

### Suggested Client Flow

```text
Requested range [start, end)
  ├── Within manifest coverage → CDN → verify → decode → scan
  └── Outside coverage / CDN failure → lightwalletd → scan
```

Cache verified index and chunk files locally when appropriate, and avoid duplicate concurrent downloads of the same chunk.

## 7. Download and Verify Example

The following script downloads the chunk containing a selected block height and verifies the index and chunk integrity. By default, it uses the current manifest's `first_height`, so no published height is hardcoded.

Requirements: `bash`, `curl`, `jq`, and either `sha256sum` (Linux) or `shasum` (macOS).

```bash
#!/usr/bin/env bash
set -euo pipefail

BASE_URL="https://zcashblocks.zknoir.com/compact-blocks/v1"
DATASET_URL="$BASE_URL/main/shielded-default"
OUT_DIR="${OUT_DIR:-./zcash-cdn-download}"
mkdir -p "$OUT_DIR"

verify_sha256() {
  if command -v sha256sum >/dev/null 2>&1; then
    sha256sum -c -
  else
    shasum -a 256 -c -
  fi
}

curl -fsSL --compressed "$DATASET_URL/manifest.json" -o "$OUT_DIR/manifest.json"
HEIGHT="${1:-$(jq -er '.first_height' "$OUT_DIR/manifest.json")}" 

jq -e --argjson h "$HEIGHT" '
  .format_version == 1 and .network == "main" and
  .profile == "shielded-default" and .pool_types == [] and
  .first_height <= $h and .published_through >= $h
' "$OUT_DIR/manifest.json" >/dev/null

INDEX_PATH=$(jq -er --argjson h "$HEIGHT" '
  .indexes[] | select(.start_height <= $h and .end_height_exclusive > $h) | .path
' "$OUT_DIR/manifest.json")
INDEX_SHA=$(jq -er --arg p "$INDEX_PATH" '
  .indexes[] | select(.path == $p) | .sha256
' "$OUT_DIR/manifest.json")

curl -fsSL --compressed "$DATASET_URL/$INDEX_PATH" -o "$OUT_DIR/index.json"
printf '%s  %s\n' "$INDEX_SHA" "$OUT_DIR/index.json" | verify_sha256

CHUNK_PATH=$(jq -er --argjson h "$HEIGHT" '
  .chunks[] | select(.start_height <= $h and .end_height_exclusive > $h) | .path
' "$OUT_DIR/index.json")
CHUNK_SHA=$(jq -er --arg p "$CHUNK_PATH" '
  .chunks[] | select(.path == $p) | .sha256
' "$OUT_DIR/index.json")
CHUNK_SIZE=$(jq -er --arg p "$CHUNK_PATH" '
  .chunks[] | select(.path == $p) | .uncompressed_size
' "$OUT_DIR/index.json")

curl -fsSL --compressed "$DATASET_URL/$CHUNK_PATH" -o "$OUT_DIR/chunk.pb"
test "$(wc -c < "$OUT_DIR/chunk.pb")" -eq "$CHUNK_SIZE"
printf '%s  %s\n' "$CHUNK_SHA" "$OUT_DIR/chunk.pb" | verify_sha256

echo "Verified chunk for height $HEIGHT: $OUT_DIR/chunk.pb"
```

Save as `download-chunk.sh` and run `bash download-chunk.sh [height]`. The output `chunk.pb` contains an entire `CompactBlockChunk`; it must be decoded using the Protobuf definition above. For requests spanning multiple indexes or chunks, repeat the selection and verification steps for every overlapping file.

## 8. Validation and Security

Before using downloaded blocks for wallet scanning, clients should:

- Verify the manifest's version, network, genesis hash, and profile against the expected chain.
- Verify index and chunk metadata, downloaded sizes, and SHA-256 hashes.
- Verify the decoded envelope's version, network, genesis hash, profile, and height interval.
- Check that every expected block height appears **exactly once**, in ascending order, and that each block's `prevHash` links to the preceding block's `hash`.
- Use trusted lightwalletd tree-state data to validate scanning boundaries. **CDN file integrity alone does not establish chain validity.**
- Treat metadata paths as dataset-relative paths; reject unexpected external URLs or path traversal.

**HTTP encoding:** JSON files are served as `application/json`; chunks as `application/x-protobuf`. Any may use `Content-Encoding: gzip`. `curl --compressed` and browser `fetch()` automatically decompress HTTP content. All size and SHA-256 checks described above apply to the **decompressed bytes**.

**Caching:** Manifest content changes over time and should be refreshed. Indexes and chunks are addressed by content hashes and are suitable for long-lived caching after validation.

## 9. Fallback Behavior

The CDN is intentionally a historical-data source, not an availability or consensus dependency. Wallets should continue syncing through lightwalletd when:

- The manifest cannot be fetched or the requested range is outside its published coverage.
- An index or chunk is missing, inaccessible, or times out.
- A size, checksum, Protobuf, network, or block-continuity check fails.
- Another required RPC method is unavailable from the CDN.

For partially covered ranges, prefer falling back **only for the missing or failed block intervals**.

## Reference

- [Zcash lightwallet protocol — compact_formats.proto](https://github.com/zcash/lightwallet-protocol/blob/main/walletrpc/compact_formats.proto)
- [Mainnet CDN manifest](https://zcashblocks.zknoir.com/compact-blocks/v1/main/shielded-default/manifest.json)
- [Cloudflare R2 — How R2 works](https://developers.cloudflare.com/r2/how-r2-works/)
- [Cloudflare R2 pricing](https://developers.cloudflare.com/r2/pricing/)
- [Cloudflare R2 public buckets and caching](https://developers.cloudflare.com/r2/buckets/public-buckets/)
- [Cloudflare R2 consistency model](https://developers.cloudflare.com/r2/reference/consistency/)
- [Cloudflare Cache for R2](https://developers.cloudflare.com/cache/interaction-cloudflare-products/r2/)
