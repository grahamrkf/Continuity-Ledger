# Continuity Ledger

**Inspect. Compare. Reconcile.**

Continuity Ledger is a planned desktop application for inspecting, comparing, auditing, and managing video files and their internal tracks across a media library.

It is intended for people maintaining sizeable Jellyfin, Plex, Emby, or other media collections who want a clear, spreadsheet-like view of their files and a safe way to make consistent track-level changes.

The project is currently in the **planning / draft specification stage**. These documents describe the proposed design; they do not represent a working release.

## Highlights

- **Collection-wide comparison:** View one file per row, with columns for container, resolution, video codec, bitrate, frame rate, HDR, and other technical properties.
- **Aligned audio and subtitle tracks:** Equivalent tracks are grouped into consistent columns across files. Missing tracks remain visibly empty instead of shifting later tracks out of alignment.
- **Difference-focused inspection:** Identify variations in codecs, languages, channel layouts, track flags, and missing tracks.
- **Track management:** Plan changes to track language, title, default/forced status, commentary and hearing-impaired flags, track order, extraction, addition, and removal.
- **Safe staged changes:** No media file is modified while browsing or editing. Changes are queued, reviewed, and confirmed before processing.
- **Conservative file handling:** Prefer lossless stream-copy/remux operations. Never silently transcode media to complete a track-management task.
- **Flexible renaming:** Build filenames from configurable templates, with optional metadata-provider-assisted movie and television identification.
- **Session persistence and exports:** Save inspection sessions and export detailed collection information to CSV, XLSX, and JSON.
- **Broad inspection, capability-aware editing:** Inspect as many media formats as supported by the available tools; editing options are enabled only when the container and processing backend can safely support them.

## Planned editing scope

The initial scope is deliberately focused on track and container management:

- Change track language and track name.
- Set default and forced flags.
- Set commentary and hearing-impaired flags.
- Reorder, remove, add, and extract tracks.
- Rename files using configurable templates.

Video and audio stream properties are not edited, and Continuity Ledger is not intended to be a transcoder.

Stream-copy remuxing may be used when a container must be rewritten to add, remove, or reorder tracks. The underlying audio, video, and subtitle streams must not be re-encoded as part of these operations.

Subtitle text editing, synchronization, translation, arbitrary audio conversion, and video conversion are outside the initial scope.

## Proposed technology

- **Python** — application and business logic
- **PySide6 / Qt** — cross-platform desktop interface
- **MediaInfo and FFprobe** — media inspection
- **MKVToolNix and FFmpeg** — supported container and stream operations
- **SQLite** — sessions and local application data
- **openpyxl** — XLSX export
- **pytest** — automated testing

Linux is the primary development and initial testing platform. Windows support is a requirement, while macOS support is intended to be straightforward to add.

## Repository layout

```text
continuity-ledger/
├── README.md
├── LICENSE
├── .gitignore
└── docs/
    ├── SDD.md
    └── TDD.md
```

## Documentation

- [Software Design Document](docs/SDD.md)
- [Technical Design Document](docs/TDD.md)

## Project status

**Draft / pre-implementation.**

The Software Design Document (SDD) and Technical Design Document (TDD) establish the intended product behavior and proposed architecture.

Implementation details, tool-version requirements, packaging choices, provider access terms, and format-specific operation support will be validated during development.

## License

Continuity Ledger is intended to be released under the **MIT License**. See [LICENSE](LICENSE) for details.
