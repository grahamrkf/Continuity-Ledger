# Continuity Ledger
## Technical Design Document (TDD)

**Document version:** 1.0  
**Project status:** Initial technical specification  
**Implementation language:** Python  
**GUI framework:** PySide6 / Qt  
**Primary development platform:** Linux

---

## 1. Technical Overview

### 1.1 Proposed Technology Stack

| Component | Technology | Purpose |
|---|---|---|
| Programming language | Python 3 | Application implementation |
| GUI | PySide6 | Cross-platform interface |
| Table framework | Qt Model/View | Large comparison tables |
| Media inspection | MediaInfo | Detailed media metadata |
| Secondary inspection | FFprobe | Stream and container inspection |
| Container editing | MKVToolNix | MKV track management |
| General media processing | FFmpeg / FFprobe | Inspection, extraction, and supported stream-copy operations |
| Database | SQLite | Sessions, configuration, and operation history |
| Data validation | Pydantic | Internal data structures and validation |
| Spreadsheet export | openpyxl | XLSX generation |
| CSV export | Python csv module | CSV generation |
| HTTP/API | httpx | Metadata-provider communication |
| Testing | pytest | Automated tests |
| Packaging | PyInstaller or Nuitka | Application packaging |

Dependencies should be selected and pinned deliberately.

The application should not require every media utility to be installed in order to inspect files. Optional tools should enable additional editing capabilities.

### 1.2 Architectural Principles

- GUI-independent business logic.
- Media-tool abstraction.
- Modular metadata providers.
- Explicit capability detection.
- Non-destructive editing.
- Transaction-based operation processing.
- Cross-platform filesystem abstraction.
- Persistent sessions and operation history.

---

## 2. Application Architecture

### 2.1 High-Level Architecture

The application should be divided into six major layers.

**Presentation layer**
- Main window.
- Comparison table.
- Properties dialogs.
- Track editors.
- Renaming interface.
- Review screen.
- Settings.

**Application layer**
- File import controller.
- Scan coordinator.
- Comparison coordinator.
- Editing coordinator.
- Renaming coordinator.
- Session manager.

**Domain layer**
- Media file model.
- Stream and track models.
- Matching engine.
- Difference detection.
- Filename template engine.
- Pending operation model.

**Infrastructure layer**
- MediaInfo adapter.
- FFprobe adapter.
- MKVToolNix adapter.
- FFmpeg adapter.
- Filesystem operations.
- Database repository.
- Metadata-provider clients.

**Processing layer**
- Background task workers.
- Command execution.
- Temporary file management.
- Output validation.
- Transaction orchestration.

**Persistence layer**
- SQLite database.
- Saved sessions.
- User settings.
- Operation history.

### 2.2 Suggested Project Structure

```text
continuity-ledger/
├── pyproject.toml
├── README.md
├── LICENSE
├── requirements/
│   ├── base.txt
│   ├── development.txt
│   └── packaging.txt
├── src/
│   └── continuity_ledger/
│       ├── __init__.py
│       ├── main.py
│       │
│       ├── ui/
│       │   ├── main_window.py
│       │   ├── comparison_view.py
│       │   ├── file_properties.py
│       │   ├── track_editor.py
│       │   ├── batch_editor.py
│       │   ├── renaming_view.py
│       │   ├── review_view.py
│       │   └── settings_view.py
│       │
│       ├── models/
│       │   ├── media_file.py
│       │   ├── media_stream.py
│       │   ├── track.py
│       │   ├── comparison.py
│       │   ├── session.py
│       │   └── operation.py
│       │
│       ├── services/
│       │   ├── scanner.py
│       │   ├── inspector.py
│       │   ├── matcher.py
│       │   ├── difference.py
│       │   ├── editor.py
│       │   ├── extractor.py
│       │   ├── renamer.py
│       │   ├── templates.py
│       │   └── session_manager.py
│       │
│       ├── adapters/
│       │   ├── mediainfo.py
│       │   ├── ffprobe.py
│       │   ├── mkvtoolnix.py
│       │   └── ffmpeg.py
│       │
│       ├── providers/
│       │   ├── base.py
│       │   ├── tvdb.py
│       │   └── tmdb.py
│       │
│       ├── database/
│       │   ├── schema.py
│       │   └── repository.py
│       │
│       ├── workers/
│       │   ├── scan_worker.py
│       │   └── process_worker.py
│       │
│       └── utils/
│           ├── paths.py
│           ├── processes.py
│           ├── validation.py
│           └── logging.py
│
├── tests/
│   ├── unit/
│   ├── integration/
│   ├── fixtures/
│   └── platform/
│
├── packaging/
│   ├── linux/
│   └── windows/
│
└── docs/
    ├── SDD.md
    └── TDD.md
```

This structure is a starting point, not a requirement to create every module immediately.

---

## 3. Media Inspection

### 3.1 Inspection Strategy

Use MediaInfo as the primary source of human-readable technical metadata.

Use FFprobe as a secondary source for:
- Stream identification.
- Container information.
- Technical properties.
- Stream ordering.
- Detailed disposition flags.
- Validation of processing results.

The application should normalize information from both tools into a common internal model.

Information from the original inspection tools should also be retained where useful, so that less common properties are not discarded.

### 3.2 Inspection Workflow

1. User selects files or folders.
2. File scanner identifies candidate media files.
3. Files are queued for inspection.
4. MediaInfo and FFprobe inspect files using background workers.
5. Results are normalized.
6. The internal media database is updated.
7. Comparison and track matching are performed.
8. The UI model is refreshed.

The inspection pipeline should continue when one tool fails, provided the other can return useful information.

### 3.3 Format Capabilities

Each file should receive an inspection status and an editing capability profile.

Example:

```python
class MediaCapabilities:
    can_inspect: bool
    can_change_track_metadata: bool
    can_add_tracks: bool
    can_remove_tracks: bool
    can_reorder_tracks: bool
    can_extract_tracks: bool
    can_modify_container: bool
    limitations: list[str]
```

Capabilities should be determined from the actual container, streams, installed utilities, and supported operations.

Do not assume that every file inspected by FFmpeg can also be safely modified by FFmpeg.

---

## 4. Internal Data Models

### 4.1 Media File

A media file should include:

```python
class MediaFile:
    id: str
    path: str
    filename: str
    extension: str
    container: str | None
    file_size: int | None
    duration: float | None
    overall_bitrate: int | None
    modified_time: float | None
    content_fingerprint: str | None
    streams: list
    capabilities: MediaCapabilities
```

The file ID should be stable within a saved session.

File paths and metadata must be revalidated before applying operations.

### 4.2 Media Stream

A common stream model should include:

```python
class MediaStream:
    id: str
    file_id: str
    stream_index: int
    stream_type: str
    codec: str | None
    language: str | None
    title: str | None
    bitrate: int | None
    channels: int | None
    channel_layout: str | None
    width: int | None
    height: int | None
    frame_rate: float | None
    is_default: bool
    is_forced: bool
    is_commentary: bool
    is_hearing_impaired: bool
    extra_metadata: dict
```

Fields that are not applicable to a stream type should remain null.

The model must support additional metadata without requiring a database migration for every newly discovered media property.

### 4.3 Track Identity

Track identity must be separate from technical properties and disposition flags.

A track identity should describe the track's purpose.

Example:

```python
class TrackIdentity:
    stream_type: str
    language: str | None
    normalized_title: str | None
    role: str | None
    channels: int | None
    subtitle_role: str | None
```

Matching should use weighted evidence rather than a single exact comparison.

Codec, bitrate, default status, and forced status should be treated as comparison properties, not mandatory identity fields.

### 4.4 Matching Engine

The matching engine should:

1. Partition streams by type.
2. Normalize languages and common track names.
3. Identify special roles such as commentary.
4. Compare channel configurations where applicable.
5. Compare subtitle roles and characteristics.
6. Calculate match confidence.
7. Assign tracks to comparison groups.
8. Flag ambiguous matches for review.

Use a configurable scoring system.

An initial scoring model might assign weights to:
- Track type.
- Language.
- Role/title.
- Channel configuration.
- Subtitle characteristics.

Codec and bitrate should contribute to difference reporting, not determine identity.

Default and forced flags must never be used as identity criteria.

When multiple tracks are plausible matches, the application should avoid silently forcing a match. The user should be able to correct the grouping manually.

The matching system should be covered by tests using files with different codecs, languages, channel layouts, duplicate tracks, and missing tracks.

---

## 5. Comparison Table Implementation

### 5.1 Qt Model/View

Use Qt's model/view architecture.

Recommended components:
- `QTableView`
- Custom `QAbstractTableModel`
- Custom `QSortFilterProxyModel`
- Custom `QStyledItemDelegate`

Do not create a separate widget for every cell.

The table model should reference normalized media and comparison objects.

### 5.2 Column Model

Use a dynamic column definition.

Example:

```python
class ComparisonColumn:
    id: str
    label: str
    column_type: str
    track_group_id: str | None
    property_name: str | None
```

Column types might include:
- File property.
- Video property.
- Audio track group.
- Subtitle track group.

This permits users to configure column visibility and ordering without modifying the underlying data.

### 5.3 Cell Rendering

Track cells should render a concise summary.

Example:

```text
English (EN) ★ 2ch AAC
```

The delegate should:
- Display track name and language.
- Display default and forced indicators.
- Display commentary/accessibility indicators.
- Display channel configuration.
- Display codec.
- Indicate differences.
- Indicate missing tracks.

Additional properties should be available in a tooltip and the detailed properties editor.

### 5.4 Interaction

Selection should be routed through the application controller.

Row-header selection opens the file-properties view.

Column-header selection opens batch actions.

Cell selection opens individual track properties.

The table should support keyboard navigation, multi-selection, and context menus.

---

## 6. Editing Engine

### 6.1 Operation Abstraction

All changes should be represented as operation objects.

Example:

```python
class PendingOperation:
    id: str
    operation_type: str
    source_file: str
    target_stream: str | None
    parameters: dict
    original_values: dict
    proposed_values: dict
    output_strategy: str | None
    status: str
```

Operation types include:
- Set language.
- Set track name.
- Set default flag.
- Set forced flag.
- Set commentary flag.
- Set hearing-impaired flag.
- Remove track.
- Reorder tracks.
- Extract track.
- Add track.
- Rename file.

The operation model should be independent of the actual media utility used to perform it.

### 6.2 Backend Selection

Create a common adapter interface for editing utilities.

The editing coordinator should select a backend based on:
- Container.
- Stream type.
- Requested operation.
- Backend availability.
- Known operation limitations.

MKVToolNix should be the preferred backend for MKV container editing.

FFmpeg may be used for supported stream-copy and extraction operations.

No backend may silently fall back to transcoding.

### 6.3 MKV Editing

For MKV files, investigate and use:
- `mkvmerge` for remuxing and stream ordering.
- `mkvpropedit` for supported in-place metadata modifications.
- `mkvextract` for track extraction.

Prefer metadata-only editing where possible.

Operations requiring remuxing should use stream copying and must preserve original stream contents.

### 6.4 Track Addition

External tracks should be inspected before they are added.

The user should be able to configure:
- Track language.
- Track title.
- Default flag.
- Forced flag.
- Commentary flag.
- Hearing-impaired flag.

The application should identify incompatible containers or stream types before attempting processing.

### 6.5 Extraction

Extraction should preserve the original stream whenever the output format permits.

If an extraction requires conversion to produce a usable output, the application should report that limitation and not perform conversion automatically.

### 6.6 Operation Validation

Before processing:
- Verify the source exists.
- Verify the source has not changed unexpectedly.
- Verify the target stream still exists.
- Verify the operation is supported.
- Verify output paths are valid.
- Verify available storage where practical.

After processing:
- Inspect the output.
- Verify expected streams.
- Verify expected metadata.
- Verify that unchanged streams remain present.
- Record the result.

---

## 7. Transaction and File Safety System

### 7.1 Pending Operation Queue

The application should maintain a persistent or recoverable queue of proposed operations.

Edits in the UI should update this queue, not immediately execute commands.

Users should be able to:
- Review all operations.
- Edit proposed values.
- Remove operations.
- Exclude individual files.
- Revert pending changes.
- Save a session without applying changes.

### 7.2 Review Process

The review screen should show the original and proposed states.

It should group operations by file and allow users to inspect individual changes.

The review must include:
- Track metadata changes.
- Track additions and removals.
- Track reordering.
- Extraction destinations.
- Renaming changes.
- Output conflicts.

### 7.3 Output Strategies

Implement three strategies.

**Overwrite original**
- Create temporary output.
- Validate output.
- Preserve or create backup.
- Replace original only after validation.

**Append filename**
- Create a copy in the same directory.
- Add a configurable suffix to the filename.
- Never overwrite an existing destination without explicit approval.

**Separate output directory**
- Preserve the source directory structure where appropriate.
- Support changed-files-only output.
- Support all-files output.
- Avoid unnecessary reprocessing of unchanged files.

### 7.4 Transaction Management

A batch may contain multiple files.

Each file should be processed as an independent unit with its own temporary output and validation.

A failed file must not cause other files to be silently discarded.

The application should maintain a manifest describing:
- Planned operations.
- Completed operations.
- Failed operations.
- Backup locations.
- Output locations.
- Validation results.

A recovery mechanism should allow users to understand and resume or clean up interrupted operations.

---

## 8. Filename Template Engine

### 8.1 Template Syntax

Use a simple, documented placeholder syntax.

Example:

```text
{series} ({year}) - S{season:02}E{episode:02} - {episode_title}
[{source}-{resolution}][{audio_codec} {channels}][{video_codec}]
```

The engine should support:
- Variables.
- Optional sections.
- Padding.
- Conditional fields.
- Sanitization.
- Missing-value handling.

The syntax should be intentionally simpler than a full programming language.

### 8.2 Metadata Sources

Filename values should be populated from:
1. Media inspection.
2. Original filename parsing.
3. Directory structure.
4. External metadata providers.
5. User-entered corrections.

Each value should retain its source and confidence where relevant.

For example, source and release group information may only be available from the original filename.

The application must not invent release information based on codec or resolution.

### 8.3 Naming Presets

Provide configurable presets inspired by:
- TRaSH Guides standard naming.
- TRaSH Guides P2P/Scene naming.
- Simple media-server naming.
- Custom user templates.

Templates should be stored separately from the application code.

### 8.4 Filename Collision Detection

Before applying renaming:
- Normalize destination paths.
- Detect duplicate filenames.
- Detect existing destination files.
- Detect invalid characters.
- Detect filesystem case-sensitivity conflicts.
- Detect conflicting operations involving multiple files.

Renames should be planned before execution.

For complex batches involving filename swaps or circular renames, use temporary intermediate names to avoid collisions.

---

## 9. Metadata Provider Architecture

### 9.1 Provider Interface

Create a common provider interface.

```python
class MetadataProvider:
    def search_series(self, query: str):
        ...

    def get_series(self, provider_id: str):
        ...

    def get_episodes(self, provider_id: str, season_order: str):
        ...

    def search_movie(self, query: str):
        ...

    def get_movie(self, provider_id: str):
        ...
```

The interface should be adapted to each provider's actual API capabilities.

### 9.2 Initial Providers

Implement provider adapters for:
- TheTVDB.
- TMDB.

IMDb should remain a potential future integration, subject to API availability and licensing.

API credentials should be stored securely using the operating system's credential facilities where available.

Provider usage must respect rate limits, attribution, and licensing requirements.

### 9.3 Episode Matching

The episode matching workflow should combine:
- Filename parsing.
- Folder names.
- Existing embedded metadata.
- Provider search results.
- User confirmation.

The user should be able to choose episode order:
- Aired.
- DVD.
- Absolute.
- Provider-supported alternatives.

Episode identification should be stored in the session so it does not need to be repeated unnecessarily.

---

## 10. Database and Persistence

### 10.1 Database

Use SQLite for local application data.

Media files themselves should not be stored in the database.

### 10.2 Initial Tables

Suggested tables:

**sessions**
- id
- name
- created_at
- updated_at
- settings_json

**session_files**
- id
- session_id
- path
- filename
- file_size
- modified_time
- fingerprint
- inspection_status
- metadata_json

**streams**
- id
- file_id
- stream_index
- stream_type
- metadata_json

**track_groups**
- id
- session_id
- track_type
- identity_json
- matching_confidence

**track_group_members**
- group_id
- stream_id

**pending_operations**
- id
- session_id
- operation_type
- source_file_id
- target_stream_id
- parameters_json
- status

**metadata_matches**
- id
- file_id
- provider
- provider_id
- match_type
- match_confidence
- metadata_json

**operation_history**
- id
- session_id
- operation_id
- timestamp
- result
- log_location

Schema migrations should be supported from the beginning.

### 10.3 Session Integrity

On reopening a session:
- Check whether source files still exist.
- Compare size and modification time.
- Reinspect files when necessary.
- Invalidate stale stream references.
- Preserve user matching decisions when they remain applicable.

A session must not blindly apply old operations to changed files.

---

## 11. Background Processing

Media inspection and file operations must not block the GUI thread.

Use Qt's worker facilities, such as `QThreadPool` and `QRunnable`, or carefully managed worker objects.

Tasks should report:
- Progress.
- Current file.
- Completed count.
- Failed count.
- Cancellation status.

Cancellation must not leave partially written files mistaken for valid outputs.

Long-running external processes should be managed through a subprocess abstraction.

The subprocess abstraction should:
- Avoid shell interpolation.
- Pass arguments as separate command-line arguments.
- Capture standard output and standard error.
- Support cancellation.
- Handle process timeouts where appropriate.
- Record exit codes.
- Sanitize sensitive information in logs.

---

## 12. Export System

### 12.1 CSV

Use the Python standard library.

Support:
- One row per file.
- Track information in separate columns.
- Optional expanded track-level exports.

### 12.2 XLSX

Use openpyxl.

Provide:
- A file overview worksheet.
- Audio tracks worksheet.
- Subtitle tracks worksheet.
- Video streams worksheet.
- Optional differences worksheet.

Preserve the underlying information even when the visible application table groups tracks together.

### 12.3 JSON

Provide a structured export containing:
- File properties.
- Stream metadata.
- Track groups.
- Differences.
- Optional session information.

Exported reports should be independent of the application's internal database schema.

---

## 13. Logging and Error Handling

Use Python's standard logging framework.

Log:
- Inspection failures.
- Unsupported formats.
- Missing dependencies.
- Metadata-provider failures.
- Processing operations.
- Validation failures.
- Output conflicts.

Errors should be presented in understandable language, with technical details available for troubleshooting.

A failed inspection should not prevent other files from being inspected.

A failed processing operation must never be reported as successful merely because an output file exists.

---

## 14. Security and Privacy

- Do not execute shell commands using untrusted filename interpolation.
- Validate all filesystem paths.
- Treat metadata and filenames as untrusted input.
- Protect API credentials.
- Do not transmit media files to metadata providers.
- Send only the minimum information necessary for provider searches.
- Do not upload media content without explicit user authorization.
- Avoid exposing credentials in logs.

---

## 15. Testing Strategy

### 15.1 Unit Tests

Test:
- Metadata normalization.
- Track matching.
- Difference detection.
- Filename template parsing.
- Filename sanitization.
- Operation planning.
- Capability detection.

### 15.2 Integration Tests

Use small media fixtures to test:
- MediaInfo inspection.
- FFprobe inspection.
- MKVToolNix editing.
- Track extraction.
- Track addition.
- Track removal.
- Track reordering.
- Metadata changes.
- Output validation.

### 15.3 Safety Tests

Verify:
- Original files remain untouched during inspection.
- Pending changes do not modify files.
- Failed processing preserves originals.
- Destination collisions are detected.
- Incomplete outputs are not accepted.
- Changed source files invalidate stale operations.
- Interrupted batch operations can be recovered safely.

### 15.4 Platform Tests

Linux should receive the most extensive initial testing.

Windows must be tested before its first supported distribution.

macOS-specific dependencies and filesystem assumptions should be avoided where possible, with macOS testing introduced when that platform is targeted.

---

## 16. Packaging and Release Engineering

### 16.1 Linux

Initial distribution:
- AppImage for 0.0.x releases.

Subsequent distribution:
- DEB packages for 0.x.0 releases.

RPM packaging may be added later.

### 16.2 Windows

Provide a standalone application or installer.

External media utilities should either be bundled where licensing permits or detected and configured separately.

### 16.3 macOS

Keep the codebase compatible with macOS conventions.

Avoid Linux-only APIs in core functionality.

A signed and packaged macOS application can be introduced later.

### 16.4 Versioning

Use semantic-versioning principles.

The proposed release pattern should be interpreted as:
- 0.0.x: early development and Linux AppImage releases.
- 0.x.0: feature milestones and additional platform packages.
- 1.0.0: stable release with a defined supported feature set.

The release pipeline should build and test packages automatically where practical.

---

## 17. Implementation Roadmap

### Milestone 1 — Application Foundation
- Project structure.
- PySide6 application.
- Dependency detection.
- File and folder importing.
- MediaInfo and FFprobe integration.
- Normalized media models.

### Milestone 2 — Comparison Interface
- Qt table model.
- File rows.
- Video property columns.
- Track matching.
- Track group columns.
- Difference highlighting.
- File and track properties.

### Milestone 3 — Editing Framework
- Pending operation model.
- Individual track editing.
- Batch editing.
- Track removal.
- Reordering.
- Addition and extraction.
- MKVToolNix adapter.

### Milestone 4 — Safe Processing
- Review screen.
- Output strategies.
- Temporary file processing.
- Validation.
- Backups.
- Operation logs.

### Milestone 5 — Renaming
- Filename parser.
- Template engine.
- Technical metadata variables.
- Preview.
- Collision detection.
- Batch renaming.

### Milestone 6 — Metadata Providers
- TheTVDB adapter.
- TMDB adapter.
- Series and movie search.
- Episode identification.
- Alternate season ordering.
- Manual matching.

### Milestone 7 — Persistence and Exports
- SQLite sessions.
- Session restoration.
- CSV export.
- XLSX export.
- JSON export.

### Milestone 8 — Distribution
- Linux AppImage.
- Linux DEB.
- Windows package.
- Automated tests and release builds.

---

## 18. Technical Decisions Requiring Validation During Implementation

The following should be treated as implementation research tasks rather than assumed capabilities:

1. Exact editing support for each container and stream combination.
2. Which operations can use metadata-only editing and which require remuxing.
3. Preservation of stream metadata and attachments during remuxing.
4. Safe extraction formats for different audio and subtitle codecs.
5. Windows availability and packaging of MKVToolNix and FFmpeg utilities.
6. TheTVDB and TMDB API access, pricing, and licensing requirements.
7. The exact TRaSH-inspired template fields and their correspondence to information recoverable from files.
8. Large-library performance and track-matching accuracy.

The application must expose limitations rather than silently performing unsupported conversions.

---

## 19. Initial Technical Success Criteria

The first functional prototype should demonstrate:

- Importing multiple MKV files.
- Extracting metadata through MediaInfo and FFprobe.
- Displaying files in a Qt comparison table.
- Aligning matching audio and subtitle tracks.
- Highlighting differences.
- Opening individual file and track properties.
- Staging a track metadata change.
- Showing the proposed change in a review screen.
- Applying the change through MKVToolNix.
- Validating the output without re-encoding the underlying streams.

This prototype should establish the core architecture before the renaming and metadata-provider systems are implemented.

**End of Technical Design Document.**
