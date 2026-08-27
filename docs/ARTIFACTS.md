# Artifact Specification — Itqan CMS (Draft)

Status: Draft — for discussion
Related: ITQ-26 (define packaged form for AssetVersion)

Purpose
-------
This document defines a minimal, practical packaged artifact contract for AssetVersion so the registry, installer, and updater can interoperate predictably. The goal is a clear, reviewable specification that implementers can target before any code changes.

Design goals
------------
- Deterministic: same AssetVersion always yields the same artifact identity (when content unchanged)
- Verifiable: manifest + checksums to validate integrity
- Idempotent installs and updates: installer can skip unchanged files
- Simple to implement: use widely-supported formats and SHA-256 checksums
- Backwards-friendly: integrates with existing models (Asset, AssetVersion, Distribution)

High-level overview
-------------------
- Artifact owner: an artifact represents one AssetVersion.
- Artifact identity: artifact_id is derived from asset_slug and version (and optional build metadata);
  the artifact file also carries a package-level checksum (sha256).
- Distribution mapping: PACKAGE channel maps to artifacts (one or more artifacts per AssetVersion).
- Packaging format: ZIP archive (artifact) + in-archive manifest.json. On-disk layout mirrors archive contents.
- Checksum algorithm: SHA-256 for files and the whole archive.
- Build mode: default is build-on-publish (produce artifact at publish time), with optional on-demand generation.

Naming & storage conventions
---------------------------
Archive name convention (registry storage / S3 / R2 path):

```text
artifacts/<asset_slug>/<version>/itqan-<asset_slug>-<version>.zip
```

Example:

```text
artifacts/recitation-saad-al-ghamidi/1.2.0/itqan-recitation-saad-al-ghamidi-1.2.0.zip
```

When unpacked on-disk (installer target):

```text
assets/<asset_slug>/<version>/
├── manifest.json
└── data/
    └── ... (media and auxiliary files) ...
```

Manifest (manifest.json)
------------------------
Location: top-level of the archive and on-disk under `assets/<asset_slug>/<version>/manifest.json`

Schema (JSON Schema / illustrative):

```json
{
  "artifact_format": "itqan-package-v1",
  "asset_id": 123,
  "asset_slug": "recitation-saad-al-ghamidi",
  "version": "1.2.0",
  "distribution_channel": "PACKAGE",
  "build_mode": "on_publish",
  "producer": {
    "service": "cms-publisher",
    "id": "web-42",
    "created_at": "2026-08-24T09:00:00Z"
  },
  "package_sha256": "<sha256-of-whole-zip>",
  "files": [
    {
      "path": "data/surah-001.mp3",
      "size_bytes": 2456789,
      "sha256": "<sha256-of-file>",
      "mime_type": "audio/mpeg"
    },
    {
      "path": "data/metadata.json",
      "size_bytes": 2345,
      "sha256": "<sha256-of-file>",
      "mime_type": "application/json"
    }
  ],
  "metadata": {
    "license": "CC-BY",
    "language": "ar",
    "notes": "optional free-text"
  }
}
```

Notes:
- artifact_format: versioned identifier for the package contract
- package_sha256: SHA-256 of the archive bytes (helpful when registry serves a single file)
- files[].sha256: SHA-256 of each file's bytes as stored inside the archive
- paths are forward-slash delimited and relative to the package root

Why both per-file and package checksums?
- package-level checksum verifies the downloaded archive as a unit
- per-file checksums let the installer decide which files changed (idempotent updates) and detect corruption after extraction

Installer behavior (algorithm)
------------------------------
Given a resolved artifact reference (registry returns artifact URL + manifest or manifest path):

1. Fetch artifact-level metadata from registry (manifest or manifest URL + package checksum).
2. If the registry returns package_sha256:
   - If installer has an existing archive file with same sha256, skip download and proceed to extraction/verification.
   - Else download the archive and verify package_sha256.
3. Extract archive to a temporary directory (do NOT overwrite live assets yet).
4. For each file listed in manifest.files:
   - If target file exists at `assets/<asset_slug>/<version>/<path>`:
     - Compute local sha256 and compare to manifest.files[].sha256
     - If equal: skip writing this file
     - Else: move/replace file from temp extraction to final path
   - If target file missing: move file from temp extraction to final path
5. After all files processed, atomically move manifest.json into place (or write a version stamp file) so the install is considered complete.
6. If any integrity check fails (download checksum mismatch, file checksum mismatch after extraction), abort and leave existing installation untouched; report error to user and registry logs.

Idempotency and partial updates
- Installer only writes files that changed according to per-file checksums.
- This allows network-efficient updates and resilience to partial failures.

Registry responsibilities
-------------------------
- Store artifacts at stable URLs under `artifacts/<asset_slug>/<version>/...`
- Expose an API to resolve an AssetVersion to one or more artifact URLs and return manifest or manifest URL + package_sha256
- Optionally support range requests/streaming for large archives
- Ensure artifact immutability once published (or provide explicit artifact re-publish semantics)

Mapping to Distribution model (apps/content/models.py)
-----------------------------------------------------
- Distribution.channel == PACKAGE indicates the AssetVersion has at least one artifact registered.
- Suggested Distribution additions (separate follow-up migration):
  - artifact_path (string, optional): canonical path/URL to artifact in registry
  - artifact_checksum (string, optional): package-level sha256
  - artifact_format (string, optional): e.g., itqan-package-v1

Build pipeline / producer
-------------------------
- Recommended default: build artifacts at publish time
  - On AssetVersion publish event, start a Celery build job:
    1. Create a temporary artifact directory
    2. Collect all files referenced by the AssetVersion (media, generated JSON, metadata)
    3. Generate data/ layout and auxiliary metadata files
    4. Produce manifest.json with per-file sha256 and package metadata
    5. Create ZIP archive from the artifact directory
    6. Compute package_sha256
    7. Upload archive to registry storage (R2 / S3 / local store)
    8. Create/Update Distribution record linking asset_version -> artifact_path + artifact_checksum
- Optional on-demand generation: registry or a worker may generate artifact if missing (cache & invalidate semantics required)

Why build-on-publish?
- Guarantees artifact immutability for a given published version
- Avoids latency when a client requests the package
- Easier to debug and reproduce

Security & integrity
--------------------
- Use SHA-256 for checksums (collision-resistance, widely-supported)
- Use HTTPS for artifact downloads
- If higher security needed, sign the manifest with a publisher key (future work)

API contract (suggested endpoints)
---------------------------------
- GET /registry/assets/{asset_slug}/versions/{version}/artifacts
  - Response: JSON list of artifacts with fields: {artifact_format, artifact_path, package_sha256, manifest_url}
- GET /registry/artifacts/{artifact_id}/manifest.json
  - Response: manifest.json as defined above
- GET artifact_path (binary) — returns archive bytes

Testing & validation
--------------------
- Unit tests for manifest generation: verify file list, sizes, per-file sha256
- Integration test: full build -> archive -> upload -> registry resolve -> installer download -> install -> verify checksums
- Failure tests: corrupted archive, missing files, partial installs

Migration & incremental approach
-------------------------------
Keep the work low-risk and incremental:

1. Add docs spec (this file) and get maintainers sign-off
2. Add Distribution fields (artifact_path, artifact_checksum, artifact_format) in a small schema migration
3. Implement build job that writes artifact to local (or R2) and populates Distribution fields
4. Implement registry endpoints to resolve artifacts and serve manifest
5. Implement installer verify+install logic and tests

Open questions / discussion items
--------------------------------
- Archive format: zip (recommended) vs tar.gz (trade-offs: Windows-friendliness vs streaming efficiency)
- Per-file path normalization rules (folder tokens, trailing slashes, reserved filenames)
- Handling of very large assets (sharding, streaming, partial downloads)
- Canonicalization of asset_slug and version (who normalizes?)
- Publisher-level signing of artifacts (future security enhancement)

Appendix: example manifest (realistic)
-------------------------------------

```json
{
  "artifact_format": "itqan-package-v1",
  "asset_id": 42,
  "asset_slug": "recitation-saad-al-ghamidi",
  "version": "1.2.0",
  "distribution_channel": "PACKAGE",
  "build_mode": "on_publish",
  "producer": {
    "service": "portal-web",
    "id": "build-123",
    "created_at": "2026-08-24T09:10:00Z"
  },
  "package_sha256": "b1946ac92492d2347c6235b4d2611184f0f2c8f4e7b1a1d3a6c6f3f4b8a6e9c0",
  "files": [
    {
      "path": "data/surah-001.mp3",
      "size_bytes": 2456789,
      "sha256": "3a7bd3e2360a6b2f3e4a5b6c7d8e9f0a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6",
      "mime_type": "audio/mpeg"
    },
    {
      "path": "data/metadata.json",
      "size_bytes": 2345,
      "sha256": "8b1a9953c4611296a827abf8c9d7e6f5a4b3c2d1e0f9a8b7c6d5e4f3a2b1c0d",
      "mime_type": "application/json"
    }
  ],
  "metadata": {
    "license": "CC-BY",
    "language": "ar"
  }
}
```

