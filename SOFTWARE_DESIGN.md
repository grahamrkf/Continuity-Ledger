# Continuity Ledger — Software Design Document

**Status:** Draft  
**Version:** 0.1  
**License:** MIT

## 1. Purpose

Continuity Ledger is a standalone desktop application for inspecting, comparing, and safely managing the tracks contained in video files. Its central workflow is to compare several versions of the same title and make informed decisions about their video, audio, and subtitle streams.

The application should be useful without a media server, database server, Docker installation, or other media-management ecosystem.

## 2. Product goals

1. Make differences between multiple media files easy to understand.
2. Match equivalent tracks across files without relying solely on codec or track order.
3. Make track-level operations safer and less error-prone than manually invoking command-line tools.
4. Support both interactive desktop use and automation through a CLI.
5. Keep the analysis and operation engine reusable by other software, including Ledgarr.
6. Avoid re-encoding video or audio unless the user explicitly requests a supported processing operation.

## 3. Target users and use cases

### Target users

- Home-media collectors comparing releases, encodes, remuxes, and editions.
- Users consolidating audio and subtitle tracks from multiple versions.
- Users who want to standardize track names, languages, flags, and ordering.
- Advanced users who want repeatable CLI reports or batch workflows.

### Primary use cases

- Compare two or more files believed to represent the same movie or episode.
- Identify a version with a missing language, commentary, forced subtitle, or preferred audio format.
- Find duplicate or near-equivalent tracks.
- Determine which file has the preferred video/audio/subtitle combination.
- Extract a track from one file and add it to another.
- Normalize track metadata and flags.
- Produce a machine-readable or human-readable comparison report.

## 4. Scope

### In scope

- Folder and file selection, drag-and-drop, recursive scanning (opt-in).
- File/container and stream metadata extraction.
- Video, audio, subtitle, attachment, and chapter inventory where supported.
- Cross-file track matching and comparison.
- Manual correction of uncertain track matches.
- Track metadata and flag editing.
- Track extraction, addition, removal, and reordering where supported.
- Safe remux-based editing that avoids re-encoding unaffected streams.
- Optional generation of an additional audio track from an existing source-language track, subject to explicit policy and review.
- Session save/restore, operation history, and report export.
- CLI for scanning, comparison, and report generation.

### Out of scope for the initial product

- Full media-library management and monitoring.
- Sonarr, Radarr, Recyclarr, or media-server administration.
- Subtitle text editing, translation, synchronization, or OCR.
- General video editing or arbitrary video conversion.
- Automatic deletion of source files.
- Unattended destructive batch editing by default.
- Creating dubbed audio or translating speech into another language.

## 5. Core user experience

### 5.1 Project/session workflow

1. User opens a folder or selects files.
2. Application scans files and extracts metadata.
3. User selects which files belong in a comparison set.
4. Application groups and aligns likely equivalent tracks.
5. Comparison view displays common properties and differences.
6. User optionally corrects track matches.
7. User selects tracks or files and queues operations.
8. Review screen presents source, destination, operation, expected changes, warnings, and backup/output strategy.
9. User confirms execution.
10. Application validates outputs and records results.

### 5.2 Comparison view

- One row per file.
- General columns include filename, container, duration, video properties, and file size.
- Audio and subtitle tracks are represented in aligned track groups/columns.
- Equivalent tracks should align even when their codecs differ.
- Missing tracks remain visibly empty.
- Codec, channel count/layout, language, title, disposition flags, and relevant technical properties are available.
- Default/forced flags are properties, not track identity.
- Commentary and accessibility tracks should be distinguishable by role.
- Ambiguous matches are marked as uncertain and can be manually corrected.
- Users can select a row (file), column/group (property or corresponding track), or cell (specific track). Batch actions must clearly state which actual tracks will be affected.

Example compact track label: `English (EN) ★ 2ch AAC`, with accessible text for icons and flags.

### 5.3 Track details

A track detail panel should expose raw and normalized metadata, including stream index, codec, language tag, title, channel layout, sample rate, bitrate when known, default/forced flags, commentary/accessibility dispositions, and source tool output where useful.

### 5.4 Editing workflow

Changes are queued rather than applied immediately. The review screen must show the before/after state and any limitations. The user can choose an output strategy:

- Write to a new file beside the source.
- Write to a selected output folder.
- Replace the source only after successful output validation, with backup/rollback safeguards.

## 6. Track matching principles

Track matching is a best-effort classification problem, not an assumption that track index N in one file equals track index N in another.

Candidate signals include:

- Stream type (audio/subtitle/video).
- Language, including normalized language codes.
- Track title and role (commentary, descriptive audio, SDH/HI, forced).
- Channel count/layout for audio.
- Codec and technical properties as supporting evidence.
- Duration and other stream characteristics.

Codec differences must not automatically prevent otherwise equivalent tracks from aligning. Default and forced flags should not define track identity. Matching should provide confidence levels and explain why a match was made. Users must be able to override a match.

## 7. Supported operations

- Set language.
- Set track title/name.
- Set default and forced flags.
- Set commentary and hearing-impaired/visual-impaired dispositions where supported.
- Reorder tracks.
- Extract tracks.
- Add tracks from another file or extracted source.
- Remove tracks.
- Generate an additional audio track from an existing track using a configured downmix/encode profile.

The application must distinguish metadata-only changes, remux operations, and operations requiring decoding/re-encoding. Existing streams should be preserved unless the user explicitly selects otherwise.

## 8. Audio-generation policies

A future policy feature may identify files missing a preferred audio variant. Example: when an English 5.1 track exists but English stereo AAC does not, generate an additional English 2.0 AAC track.

Policies may specify source language, source channel eligibility, target codec, target channel layout, bitrate, default disposition, track title, and whether approval is required. The generated track must be additive by default. Never infer that a track in one language can satisfy another language's requirement. Skip generation when an equivalent target track already exists.

Audio generation is an optional processing feature, not a general transcoding system. Preserve original audio and video streams wherever possible.

## 9. Safety and data integrity

- Scanning and comparison are read-only.
- No operation occurs merely because a user edits a field in the UI.
- Every operation is represented in a pending queue and reviewed before execution.
- Revalidate source file identity and track mapping immediately before applying changes.
- Process to a temporary output; validate the output before commit.
- Preserve the original on failure.
- Provide backup or output-to-new-file options.
- Record operation parameters, tool versions, timestamps, input/output paths, and results.
- Detect insufficient disk space and destination collisions before starting.
- Never silently overwrite an existing destination.
- Warn when a format cannot preserve a requested property.
- Do not promise lossless operation when a selected action requires re-encoding.

## 10. File-format strategy

MKV/Matroska is the primary target for comprehensive track manipulation because MKVToolNix supports a broad range of track-level operations. Other containers should be inspected where FFprobe supports them, but editing capability must be determined per operation and format. The UI should show capability limitations rather than pretending every container is equally editable.

## 11. CLI requirements

The CLI should expose stable, documented commands for:

- Scanning files/folders.
- Comparing an explicit set of files.
- Exporting JSON/CSV reports.
- Inspecting available tool capabilities.
- Optionally validating a proposed operation plan.

Destructive editing commands should require explicit arguments and confirmation controls (for example, an explicit output path or `--yes` flag). The CLI should return meaningful exit codes and machine-readable output options.

## 12. Non-functional requirements

- Responsive UI while scans and operations run in background workers.
- Cancellation for scans and long-running operations where safe.
- Clear progress and error reporting.
- Deterministic, testable comparison logic.
- No network connection required for core functionality.
- Support current mainstream Linux desktop environments initially; other desktop platforms can follow.
- Accessible labels for icons and status indicators.
- Logs should avoid exposing unnecessary sensitive path or metadata details in shared diagnostics.

## 13. Success criteria

The initial release is successful when a user can select multiple MKV files, obtain reliable metadata, compare and manually correct track alignment, queue supported track edits, review the exact changes, execute them without damaging the originals on failure, and export a comparison report.
