# Continuity Ledger

**A focused desktop tool for comparing video files and managing their tracks.**

Continuity Ledger helps users understand what differs between multiple versions of a movie or episode: video streams, audio languages and formats, subtitle tracks, flags, metadata, and container details. It is designed for direct, folder-based use without requiring Jellyfin, Plex, Emby, Sonarr, Radarr, Docker, or a remote service.

## Highlights

- Open folders, individual files, or drag and drop multiple video files.
- Extract detailed stream and container metadata using FFprobe, with optional MediaInfo enrichment.
- Compare files in a table, aligning equivalent audio and subtitle tracks even when codecs differ.
- Identify missing tracks, differing languages, channel layouts, codecs, flags, and other properties.
- Edit track language/name/default/forced/commentary/hearing-impaired flags and ordering where the container and tools support it.
- Add, extract, or remove tracks; optionally generate an additional compatible audio track through a separately reviewed processing workflow.
- Review pending operations before execution, preserve originals until validation succeeds, and keep an operation history.
- Save comparison sessions and export reports.
- Offer a CLI for scripted scans, comparisons, and reports.

## What it is not

Continuity Ledger is not a media-server replacement, library manager, transcoding farm, subtitle editor, or general-purpose video converter. It does not need a media server or *Arr integration to perform its core job.

## Proposed stack

Python core, PySide6 desktop GUI, Typer CLI, SQLite session store, FFprobe, MKVToolNix, and FFmpeg. See [Software Design](SOFTWARE_DESIGN.md) and [Technical Design](TECHNICAL_DESIGN.md).
