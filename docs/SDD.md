# Continuity Ledger
## Software Design Document (SDD)

**Document version:** 1.0  
**Project status:** Initial design specification  
**Application category:** Cross-platform media inspection, auditing, and management  
**Primary platform:** Linux  
**Secondary platform:** Windows  
**Future platform:** macOS

---

## 1. Project Overview

### 1.1 Purpose

Continuity Ledger is a desktop application for inspecting, comparing, auditing, and managing video files and their internal tracks.

The application is intended to make it easy to examine large media collections, identify inconsistencies between files, and perform batch operations without requiring users to work with command-line tools.

Its primary purpose is to provide a unified interface for:

- Inspecting video, audio, and subtitle properties.
- Comparing multiple media files in a spreadsheet-style interface.
- Identifying equivalent audio and subtitle tracks across files.
- Detecting differences in codecs, bitrates, languages, resolutions, and metadata.
- Editing track metadata and managing tracks.
- Extracting and adding tracks.
- Batch renaming files using customizable naming conventions.
- Identifying television episodes and movies through external metadata providers.
- Exporting media inventories and comparison results.

Continuity Ledger is intended to be a media inspection and management tool, not a media transcoding application.

### 1.2 Design Philosophy

The application should prioritize:

1. **Clarity:** Media information should be easy to understand and compare.
2. **Consistency:** Equivalent tracks should appear in consistent columns across files.
3. **Safety:** No file modifications should occur without explicit user confirmation.
4. **Batch operations:** Common operations should be possible across many files.
5. **Transparency:** Users should be able to review every proposed modification.
6. **Compatibility:** Inspection should support as many media formats as available tools permit.
7. **Extensibility:** New formats, metadata providers, and naming schemes should be addable without redesigning the application.

### 1.3 Target Users

The application is intended for users managing personal and professional media collections, particularly:

- Home media server administrators.
- Jellyfin, Plex, and Emby users.
- Film and television collectors.
- Users maintaining large episode libraries.
- Users who want consistent audio and subtitle track configurations.
- Users who need to audit and organize large collections.

---

## 2. Scope

### 2.1 Included Features

The application will provide the following functionality.

**Media inspection**
- Inspect video files using supported media analysis tools.
- Display container, video, audio, and subtitle information.
- Display file size, duration, resolution, bitrate, frame rate, codecs, languages, and other available metadata.
- Inspect files even when editing is unsupported.
- Identify unsupported or incomplete metadata.

**Media comparison**
- Display one row per file.
- Display video properties in dedicated columns.
- Align equivalent audio and subtitle tracks into shared columns.
- Highlight differences between equivalent tracks.
- Display missing tracks without shifting other tracks.
- Support sorting, filtering, and multi-selection.

**Track management**
- Change track language.
- Change track name.
- Set default and forced flags.
- Set commentary and hearing-impaired flags.
- Remove tracks.
- Reorder tracks.
- Extract tracks.
- Add external tracks.

**File renaming**
- Generate filenames from customizable templates.
- Include technical media properties in filenames.
- Support presets inspired by TRaSH Guides.
- Identify television episodes and movies through metadata providers.
- Preview and batch-apply renaming operations.

**Data management**
- Save and reopen comparison sessions.
- Export media inventories and comparison tables.
- Maintain operation logs.
- Review and manage pending changes.

### 2.2 Excluded Features

The following are outside the intended scope:

- Video transcoding.
- Audio transcoding.
- Video codec conversion.
- Audio codec conversion.
- General video-property modification.
- Audio editing.
- Subtitle text editing.
- Subtitle synchronization and timing correction.
- Subtitle translation.
- Automated subtitle generation.

Container-level remuxing using stream copying is permitted when required for track management. This must not re-encode the underlying audio or video streams.

The application may eventually integrate with external tools for operations outside its scope, but should not attempt to recreate a complete media conversion suite.

---

## 3. Supported Platforms and Formats

### 3.1 Operating Systems

Linux is the primary development and testing platform.

Windows support is a required project objective, not an optional future feature.

macOS support should be considered during architecture and dependency selection so that it can be introduced later without major restructuring.

### 3.2 Distribution Strategy

Proposed distribution schedule:

| Release series | Platforms and packages |
|---|---|
| 0.0.x | Linux AppImage |
| 0.x.0 | Linux DEB and Windows distribution |
| Later 0.x releases | Additional packages and refinements |
| Future | macOS application bundle |

Packaging should be automated as the project matures.

### 3.3 Media Format Support

The application should inspect as many formats as MediaInfo and FFprobe support.

Examples include:

- MKV
- MP4
- M4V
- AVI
- MOV
- WebM
- M2TS
- TS
- MPEG
- WMV
- Other formats supported by the installed inspection tools.

Inspection support and editing support must be treated separately.

A file that can be inspected but not safely modified must remain available in the comparison interface and be clearly marked as read-only for unsupported operations.

MKV should be the primary target for comprehensive track editing.

---

## 4. User Interface Design

### 4.1 Main Interface

The main interface should resemble a spreadsheet or database table.

Each row represents one media file.

Columns should include:

- Filename.
- Container.
- Resolution.
- Video codec.
- Video bitrate.
- Frame rate.
- HDR information.
- Audio tracks.
- Subtitle tracks.
- Other configurable properties.

The table should support:

- Column resizing.
- Column visibility settings.
- Sorting.
- Filtering.
- Multi-row selection.
- Multi-column selection.
- Horizontal and vertical scrolling.
- Search.
- Difference highlighting.
- Configurable display density.

The interface must remain responsive when handling large collections.

### 4.2 Track Column Organization

Equivalent tracks should be aligned across files.

For example:

| File | Audio 1 | Audio 2 | Subtitle 1 |
|---|---|---|---|
| Episode 01 | English (2ch) AAC | Japanese (6ch) DTS | English SRT |
| Episode 02 | English (2ch) AC3 | Japanese (6ch) DTS | English SRT |
| Episode 03 | English (2ch) AAC | — | English ASS |

Different codecs should not automatically create separate columns when tracks otherwise serve the same purpose.

Codec differences should instead be highlighted within the matching column.

A genuinely different track, such as commentary audio, should receive a separate column.

### 4.3 Track Cell Presentation

Cells should use compact labels and visual indicators.

Example:

`English (EN) [default] [2ch] [AAC]`

Possible indicators:

- Default: star or equivalent symbol.
- Forced: flag symbol.
- Commentary: microphone or commentary indicator.
- Hearing impaired: accessibility indicator.
- Missing track: empty cell or clearly visible placeholder.

The exact symbols should be configurable or documented, and must not rely exclusively on colour.

Hovering over a cell should reveal additional information.

### 4.4 Selection Behaviour

The table uses three selection levels.

**Row header — file-level properties**

Clicking a row header opens a detailed file-properties view.

This view includes:
- General file information.
- Container information.
- Video streams.
- Audio streams.
- Subtitle streams.
- Available metadata.
- File editing capabilities.

**Column header — batch operations**

Clicking a column header selects the corresponding property or matching track across the files.

Available actions depend on the column type.

For audio and subtitle columns, actions include:
- Change language.
- Change name.
- Set or unset flags.
- Remove matching tracks.
- Extract matching tracks.
- Other applicable batch operations.

Operations must only affect files containing the matching track.

**Individual cell — track-level properties**

Clicking a cell opens an editor for that specific track.

The editor should expose all relevant properties and available operations.

Double-clicking a cell should also open its editor.

### 4.5 Context Menus

Right-clicking should provide contextual operations.

Row context menu:
- Inspect file.
- Edit file properties.
- Select file.
- Add file to selection.
- Remove file from session.

Column context menu:
- Select matching tracks.
- Batch edit.
- Extract tracks.
- Remove matching tracks.

Cell context menu:
- Inspect track.
- Edit track.
- Extract track.
- Remove track.

Only valid operations should be available.

---

## 5. Track Matching and Comparison

### 5.1 Matching Principles

Track matching must be semantic rather than based exclusively on track index or codec.

Potential matching attributes include:

- Track type.
- Language.
- Track title.
- Channel configuration.
- Commentary status.
- Hearing-impaired status.
- Subtitle characteristics.
- Other identifying metadata.

Codec, bitrate, and default/forced flags should not independently determine track identity.

Default and forced flags are properties of a track, not its identity.

### 5.2 Difference Detection

The application should distinguish between:

1. Equivalent tracks with different technical properties.
2. Tracks with different purposes.
3. Tracks missing from some files.
4. Tracks with incomplete or ambiguous metadata.

Examples:

- English AAC stereo versus English AC3 stereo: potentially equivalent tracks with a codec difference.
- English main audio versus English commentary: separate tracks.
- English subtitle versus English forced subtitle: separate roles when their purposes differ.
- Default versus non-default: same track identity, different flag.

Matching should be conservative when metadata is insufficient. Ambiguous tracks should be identified rather than silently grouped incorrectly.

### 5.3 Highlighting

Differences should be visually identifiable.

Examples:
- Different video resolution.
- Different video codec.
- Different bitrate.
- Different audio codec.
- Different channel configuration.
- Different language.
- Missing tracks.
- Different default or forced flags.

Colour should be supplemented with text or symbols for accessibility.

---

## 6. Track Editing

### 6.1 Supported Operations

The initial editing system must support:

- Change track language.
- Change track name.
- Set and unset default flags.
- Set and unset forced flags.
- Set and unset commentary flags.
- Set and unset hearing-impaired flags.
- Remove tracks.
- Reorder tracks.
- Extract tracks.
- Add tracks.

Operations can be performed individually or in batches where technically appropriate.

### 6.2 Track Extraction

The application should extract selected tracks without re-encoding them.

The user should be able to select:
- One track.
- Multiple tracks.
- A matching track across multiple files.

The application should select an appropriate output format where possible and explain limitations when a stream cannot be extracted in a useful standalone format without conversion.

### 6.3 Adding Tracks

The application should allow users to add external audio and subtitle tracks.

Adding a track may require a container-level remux operation.

No audio or video transcoding should occur.

Track metadata should be editable before applying the operation.

### 6.4 Reordering

Track reordering should allow users to change stream order within a file.

The application should preview the resulting order before processing.

---

## 7. File Renaming and Metadata Identification

### 7.1 Renaming System

The application should provide a dedicated renaming module.

Renaming must be based on customizable templates rather than fixed naming rules.

### 7.2 Template Variables

Potential variables include:

**Identification**
- Series title.
- Movie title.
- Release year.
- Season number.
- Episode number.
- Episode title.
- Absolute episode number.
- Episode range.

**Technical information**
- Resolution.
- Video codec.
- Bit depth.
- HDR format.
- Audio codec.
- Audio channel configuration.
- Audio languages.
- Subtitle languages.

**Release information**
- Source.
- Edition.
- Release group.

**File information**
- Original filename.
- Extension.
- Directory name.

Unavailable information should be omitted rather than invented.

### 7.3 Presets

Provide presets inspired by:

- TRaSH Guides standard naming.
- TRaSH Guides P2P/Scene naming.
- Simple Jellyfin/Plex-compatible naming.
- Custom user templates.

The application must not imply official TRaSH Guides affiliation or compatibility beyond the actual implemented template.

### 7.4 Metadata Providers

The provider architecture should initially consider:

- TheTVDB.
- TMDB.

IMDb may be investigated as an additional source, subject to its data access and licensing restrictions.

Providers must be modular.

Users should be able to configure provider credentials and choose their preferred provider.

### 7.5 Episode Identification

The renaming workflow should support:

- Series identification.
- Movie identification.
- Season and episode lookup.
- Episode titles.
- Alternate season orders.
- Aired order.
- DVD order.
- Absolute numbering.
- Specials and season zero.
- Multiple episodes in one file.
- Manual matching and correction.

When identification is uncertain, the application must request user confirmation.

### 7.6 Renaming Preview

The application must show original and proposed filenames side by side.

It must detect:
- Filename collisions.
- Duplicate destination paths.
- Invalid characters.
- Missing metadata.
- Ambiguous episode matches.
- Files that cannot be renamed safely.

Renaming operations must use the same pending-change and review system as track editing.

---

## 8. Change Management and File Safety

### 8.1 Pending Changes

All edits must initially be staged.

No modification to original files should occur simply because the user changed a property in the interface.

Pending changes should include:
- Target file.
- Target track, where applicable.
- Operation.
- Original value.
- Proposed value.
- Required processing operation.
- Expected output path.

### 8.2 Review Screen

Before applying changes, users must receive a complete review of proposed operations.

The review should show:
- Every affected file.
- Every affected track.
- Original values.
- Proposed values.
- Renaming changes.
- Files to be extracted or added.
- Potential conflicts and warnings.

Individual operations should be removable from the pending queue.

### 8.3 Output Options

After reviewing the changes, the user must select an output strategy:

1. Overwrite the original files.
2. Save modified files with an appended filename in the same directory.
3. Save to a separate output directory.

For output-directory mode, users must choose between:
- Changed files only.
- All files.

Unchanged files must not be unnecessarily reprocessed.

### 8.4 Backups and Validation

The application should offer backup protection before overwriting originals.

For container modifications:
- Write to a temporary output.
- Validate the output.
- Confirm that required streams are present.
- Confirm that expected metadata changes were applied.
- Only replace the original after successful validation.

Failed operations must preserve the original file.

### 8.5 Operation Logs

Maintain logs containing:
- Date and time.
- Source file.
- Destination file.
- Operation performed.
- Success or failure.
- Error details.
- Validation results.

---

## 9. Sessions and Exports

### 9.1 Saved Sessions

Users should be able to save and reopen comparison sessions.

A session should preserve:
- File list.
- Comparison settings.
- Track matching decisions.
- Column layout.
- Filters and sorting.
- Pending changes, where safe and appropriate.
- Metadata lookup decisions.

A saved session should not be treated as a substitute for checking whether source files have changed.

### 9.2 Exports

The application should support exporting comparison information.

Initial formats:
- CSV.
- XLSX.
- JSON.

The exported information should include all available file and track details, not only the columns currently visible in the interface.

The application should also provide a human-readable report suitable for reviewing a media collection.

---

## 10. Non-Functional Requirements

### Performance
- Remain responsive during large media scans.
- Use background workers for media inspection and file processing.
- Avoid creating unnecessary UI widgets for individual table cells.
- Support large collections through model/view table architecture.

### Reliability
- Handle unsupported files without crashing.
- Preserve originals on processing failure.
- Detect missing files and changed source files.
- Report partial failures clearly.

### Usability
- Avoid requiring command-line knowledge.
- Make destructive operations explicit.
- Provide clear descriptions of technical properties.
- Support keyboard navigation and accessibility.

### Portability
- Separate application logic from operating-system-specific functionality.
- Use cross-platform libraries and media tools where possible.
- Detect missing dependencies and explain how to resolve them.

### Maintainability
- Use modular architecture.
- Keep media inspection, matching, editing, renaming, and metadata-provider functionality separate.
- Provide automated tests for important operations.

---

## 11. Development Priorities

**Phase 1 — Foundation**
- Application skeleton.
- Dependency detection.
- File and folder importing.
- Media inspection.
- Main comparison table.
- Basic difference detection.

**Phase 2 — Track management**
- Track matching.
- Detailed properties.
- Individual and batch editing.
- Pending changes.
- Review screen.
- Safe output processing.

**Phase 3 — Renaming**
- Template engine.
- Technical metadata variables.
- Renaming preview.
- Collision detection.
- Batch renaming.

**Phase 4 — Metadata identification**
- TheTVDB integration.
- TMDB integration.
- Episode matching.
- Alternate episode orders.
- Manual corrections.

**Phase 5 — Persistence and distribution**
- Saved sessions.
- CSV/XLSX/JSON exports.
- Linux packaging.
- Windows packaging.
- Automated testing and release infrastructure.

---

## 12. Acceptance Criteria

The application should be considered functionally ready for its initial release when it can:

1. Import a folder containing multiple video formats.
2. Inspect supported files without modifying them.
3. Display one row per file.
4. Align equivalent audio and subtitle tracks.
5. Highlight differences between files.
6. Open file-level and track-level properties.
7. Stage individual and batch track changes.
8. Review all proposed changes before applying them.
9. Safely modify supported containers without re-encoding streams.
10. Preserve originals when processing fails.
11. Preview and apply batch filename changes.
12. Save and reopen comparison sessions.
13. Export detailed media information.
14. Operate on Linux and provide a viable Windows distribution.

**End of Software Design Document.**
