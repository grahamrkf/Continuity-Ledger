# Continuity Ledger — Technical Design Document

**Status:** Draft  
**Version:** 0.1

## 1. Architectural approach

Use a Python application core that is independent of presentation and invocation. The desktop GUI and CLI are separate adapters over the same application services. Media manipulation is delegated to external, well-established tools.

```text
                 ┌──────────────────────────┐
                 │     Application Core     │
                 │ scan / compare / match   │
                 │ plan / validate / apply  │
                 └────────────┬─────────────┘
                              │
             ┌────────────────┼────────────────┐
             ▼                ▼                ▼
       ┌───────────┐    ┌────────────┐   ┌──────────────┐
       │ PySide6   │    │ Typer CLI  │   │ Future API   │
       │ Desktop   │    │            │   │ / Ledgarr    │
       └───────────┘    └────────────┘   └──────────────┘
                              │
                 ┌────────────┴─────────────┐
                 ▼                          ▼
          ┌──────────────┐          ┌────────────────┐
          │ FFprobe /    │          │ MKVToolNix /   │
          │ MediaInfo    │          │ FFmpeg         │
          └──────────────┘          └────────────────┘
```

Do not make the core import GUI modules or depend on command-line output intended for humans. Prefer structured JSON output from tools where available.

## 2. Proposed technology

- **Language:** Python 3, with type hints.
- **Desktop GUI:** PySide6 (Qt).
- **CLI:** Typer.
- **Validation/models:** Pydantic or dataclasses, selected consistently.
- **Local persistence:** SQLite, with a small migration mechanism.
- **Media probing:** FFprobe as the primary structured probe; optional MediaInfo for supplemental fields.
- **Container editing:** MKVToolNix (`mkvmerge`, `mkvextract`, `mkvpropedit` where applicable).
- **Audio/video processing:** FFmpeg/FFprobe.
- **Testing:** pytest; fixtures containing small, redistributable sample media or generated test assets.
- **Packaging:** Python package plus platform-specific desktop packaging when mature.

Python is appropriate because the program coordinates subprocesses, file I/O, metadata normalization, comparison, and UI state. The heavy decoding, encoding, and muxing work is performed by native media tools, so rewriting the coordinator in C/C++ would not normally improve end-to-end processing performance.

## 3. Package layout

A proposed source layout:

```text
src/continuity_ledger/
├── domain/
│   ├── models.py
│   ├── languages.py
│   ├── dispositions.py
│   └── capabilities.py
├── application/
│   ├── scan.py
│   ├── compare.py
│   ├── match_tracks.py
│   ├── plan_operations.py
│   └── execute_operations.py
├── infrastructure/
│   ├── ffprobe.py
│   ├── mediainfo.py
│   ├── mkvtoolnix.py
│   ├── ffmpeg.py
│   ├── filesystem.py
│   └── database.py
├── cli/
│   └── main.py
├── gui/
│   ├── main.py
│   ├── models/
│   └── views/
└── __main__.py
```

Keep domain and application layers free of Qt types and subprocess-specific details. Infrastructure adapters translate external tool results into normalized domain objects.

## 4. Domain model

Initial entities:

- **MediaFile:** canonical path, size, modification time, optional content fingerprint, container, duration, probe status.
- **Stream:** source stream index, type, codec, language, title, dispositions, technical properties, raw metadata.
- **TrackGroup:** comparison-level grouping of streams believed to serve the same role across files.
- **TrackMatch:** mapping from a file's stream to a group, with confidence and rationale.
- **ComparisonSession:** selected files, scan snapshots, groups, manual match corrections, filters, and UI state.
- **OperationPlan:** ordered set of proposed changes, source snapshots, destination strategy, and required tools.
- **OperationResult:** execution status, logs, output validation, backup details, and timestamps.

Store raw probe output alongside normalized fields where practical so new metadata fields can be supported without losing source information.

## 5. Scanning and metadata extraction

1. Discover candidate files using configurable extensions plus optional content probing.
2. Record filesystem identity and timestamps.
3. Run FFprobe with structured JSON output.
4. Normalize streams, dispositions, language codes, chapters, and container properties.
5. Optionally enrich with MediaInfo.
6. Record tool versions and probe errors.
7. Cache results using a fingerprint based on path, size, modification time, and optionally a content hash.

Do not use filename extensions as proof of actual container format. Avoid hashing very large files by default; make full hashing optional.

## 6. Track matching engine

Use a staged matching strategy:

1. Partition by stream type.
2. Normalize language tags and role/disposition metadata.
3. Score candidate pairs using language, role, title, channel layout, duration, and supporting codec/technical information.
4. Assign high-confidence matches automatically.
5. Leave ambiguous or conflicting candidates unmatched or flagged for review.
6. Allow manual user overrides and persist them in the session.

Matching must be deterministic for identical inputs and versioned so future algorithm improvements can be explained. Do not treat stream index as identity. Do not use default/forced flags as primary identity features.

## 7. External tool adapters

### FFprobe

Use `ffprobe -v error -show_format -show_streams -of json` (with carefully selected additional fields as needed). Parse JSON, validate expected shapes, and tolerate missing/unknown metadata. Never build shell command strings from untrusted filenames; pass argument arrays to subprocess APIs.

### MediaInfo

Use its JSON output when available. Treat it as an optional enrichment source, not a hard runtime requirement.

### MKVToolNix

Use `mkvmerge -J` for Matroska identification and structured track information where appropriate. Use `mkvextract` for extraction and `mkvmerge` for remux/add/remove/reorder workflows. Evaluate `mkvpropedit` for metadata-only changes where it safely supports the requested field. Capability-test tool versions and supported operations.

### FFmpeg

Use FFmpeg for audio generation and any explicitly requested decode/encode processing. Prefer stream-copy for unaffected streams. Construct filter graphs and codec arguments from validated operation plans. Capture stderr for diagnostics, but redact secrets and avoid logging excessive media metadata.

## 8. Operation execution and transactional safety

Treat every modification as a transaction-like workflow:

1. Re-probe source and compare it with the snapshot used to create the plan.
2. Check source/destination permissions, free space, and collisions.
3. Create a temporary output in the destination filesystem where possible.
4. Execute the selected tool using an argument list (never shell interpolation).
5. Probe the temporary output and verify expected streams, metadata, duration, and stream-copy expectations.
6. Optionally compare hashes of copied source streams where the workflow permits.
7. Commit by atomic rename when supported.
8. If replacing a source, retain a backup until the operation is confirmed and the user-defined retention period expires.
9. On failure, preserve the original and clean up temporary files only when safe.
10. Record a durable operation result.

Operations should be cancellable before commit. Once a commit begins, cancellation must not leave the source in an ambiguous state. Use a per-file lock to prevent concurrent edits to the same source.

## 9. Session storage

SQLite stores session metadata, scan snapshots, manual matching decisions, operation plans/results, and preferences. Media files themselves are never copied into the database. Session files should remain usable when source files are temporarily offline, while clearly marking stale or unavailable paths.

## 10. Interfaces

### Python application interface

Expose typed service methods, for example:

- `scan_paths(paths, options) -> ScanResult`
- `compare_files(file_ids, options) -> ComparisonResult`
- `update_track_match(session_id, correction) -> ComparisonResult`
- `create_operation_plan(session_id, requested_changes) -> OperationPlan`
- `validate_operation_plan(plan_id) -> ValidationResult`
- `execute_operation_plan(plan_id, approval) -> OperationResult`

These are conceptual contracts; exact names and schemas can be finalized during implementation.

### CLI

Provide commands such as:

```bash
continuity-ledger scan /media/incoming --json
continuity-ledger compare file-a.mkv file-b.mkv --output comparison.json
continuity-ledger report session-id --format csv
continuity-ledger tools
```

CLI editing should require an explicit output/overwrite strategy and should not bypass the same operation planner and validation used by the GUI.

### Future Ledgarr integration

Prefer direct use of the Python application/service layer when both applications run in a compatible environment. If process isolation, independent deployment, or versioning requires it, expose a local authenticated API or worker protocol. Do not make Ledgarr automate the GUI or parse human-readable CLI output.

## 11. Concurrency and performance

- Run probes and media operations in worker threads/processes, not on the GUI event loop.
- Bound concurrent FFprobe processes to avoid disk thrashing.
- Use a task queue with progress, cancellation, and per-file locks.
- Cache probe results and invalidate when file identity changes.
- Avoid loading full media files into memory.
- Stream tool output and cap retained diagnostic logs.
- Profile before introducing native extensions or rewriting components in a lower-level language.

## 12. Dependency and packaging strategy

The application should detect required external binaries and their versions at startup.

- FFprobe: required for baseline scanning.
- MKVToolNix: recommended for comprehensive Matroska editing.
- FFmpeg: recommended for processing and audio generation.
- MediaInfo: optional enrichment.

The GUI should remain useful for scanning/comparison when optional editing tools are absent, while clearly disabling unsupported operations. Provide installation guidance per platform. A later Windows/macOS package may bundle tools only after reviewing each dependency's redistribution terms and license obligations. Do not assume the MIT license of this repository changes third-party licenses.

## 13. Security

- Never invoke subprocesses through a shell with interpolated user-controlled paths.
- Validate paths and operation parameters.
- Restrict file access to user-selected locations where feasible.
- Do not expose a network listener by default.
- Treat media metadata and filenames as untrusted input.
- Avoid automatic execution of embedded attachments or scripts.
- Keep backups and temporary outputs in predictable, permission-restricted locations.

## 14. Testing

- Unit tests for language normalization, matching scores, and operation planning.
- Integration tests against multiple FFprobe/MKVToolNix/FFmpeg versions.
- Test files for missing metadata, unusual language tags, duplicate tracks, commentary, forced subtitles, and mixed codecs.
- Failure injection for full disks, permission errors, interrupted subprocesses, and invalid output.
- Verify that failed operations never destroy the original.
- GUI tests for selection semantics and pending-operation review.
- Golden JSON fixtures for stable probe parsing.

## 15. Open technical decisions

- Minimum supported Python and Qt versions.
- Initial platform support (Linux-first is recommended).
- Whether MediaInfo is installed separately or included in selected packages.
- Exact track matching algorithm and confidence thresholds.
- Whether the first release persists sessions in a global SQLite database or portable project/session files.
- Which non-MKV containers receive editing support in the first release.
- Whether Ledgarr imports the Python package directly or communicates with a worker service.
