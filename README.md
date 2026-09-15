# Tdarr Flow: Tiered Intel QSV HEVC

Annotated walkthrough of `data.txt` (flow id `90Qkh5CCi`). The raw export is
left untouched — this is a companion doc, traced from `flowEdges` node by
node. Node IDs from the export are given in `code font` so you can match
each stage back to the JSON.

## Big picture

This flow re-encodes video to HEVC at a bitrate scaled to the file's own
resolution, but only when the file actually looks like it's worth
re-encoding (already-low-bitrate files, HDR sources, and tiny files are all
routed around the encoder untouched). After encoding, it checks whether the
result actually shrank the file by a sane amount — if not, it throws away
the encode, restores the original streams, and still runs the
cleanup/remux/subtitle/audio housekeeping on the *original* file so the
library stays tidy even when re-encoding wasn't worth it.

## 1. Entry gates — decide if this file is even a candidate

- **`input_file`** → **`check_hdr`**
  HDR sources are routed to a dead end (no edge is wired from `check_hdr`'s
  "HDR detected" output) — **HDR files are left alone entirely**, presumably
  to avoid stripping HDR metadata during a QSV HEVC re-encode.
- **`check_hdr`** (non-HDR) → **`check_file_size_min`** (≥50MB)
  Below 50MB is another dead end — **tiny files are skipped**.
- **`check_file_size_min`** (pass) → **`check_resolution2`**

## 2. Resolution-tiered bitrate gate

`check_resolution2` fans out by resolution bucket into four bitrate checks,
each with a different "not worth re-encoding" floor:

| Bucket | Node | Skip-encode if bitrate ≤ |
|---|---|---|
| SD | `bitrate_sd` | 3000k |
| 720p | `bitrate_hd` | 4000k |
| 1080p | `bitrate_fullhd` | 5500k |
| 4K/8K | `bitrate_4k` | 12000k |

(`check_resolution2` has 9 output handles; handles 1–2 both route to
`bitrate_sd`, 3→`bitrate_hd`, 4→`bitrate_fullhd`, and 5–9 all funnel into
`bitrate_4k` — the exact resolution each handle number represents isn't
something I've verified against the plugin's source, but the bucketing by
threshold is clear from the wiring.)

Each bitrate check has three outcomes:
- **Above threshold** → `check_mp4` (worth re-encoding, continue pipeline)
- **At/below threshold** → `set_skip_encode_early` → `check_mp4` (still runs
  cleanup/remux, but flags `skipEncode = TRUE` so the FFmpeg stage is
  bypassed later)
- **Probe error (`err1`)** → `reset_flow_error2` → `check_mp4` (fails open —
  treated the same as "worth re-encoding" rather than silently dropping the
  file)

## 3. Container & codec triage

- **`check_mp4`**: `.mp4` → **`remux_mp4`** (remuxes to MKV via the MC93
  Remux plugin) → `check_hevc`. Non-mp4 skips straight to `check_hevc`.
- **`check_hevc`** → **`check_vp9`** → **`check_av1`**: files already in
  HEVC, VP9, or AV1 skip straight to `remove_attachments` (no reason to
  re-verify codec-appropriateness). Anything else (e.g. H.264) falls through
  to one more gate:
- **`check_video_bitrate`** (skip if ≤1000k): guards against re-encoding
  already-tiny-bitrate H.264 — below 1000k is a dead end (left alone,
  matching the HDR/tiny-file pattern above). A probe error here
  (`reset_flow_error`) fails open to `remove_attachments`.

## 4. Stream cleanup (runs on every surviving file, encode or not)

`remove_attachments` (strip image/attachment streams) →
`reorder_streams_pre` → `clean_subtitles` (keep `eng`/`und`, tag
commentary) → `clean_audio` (keep `eng,und,jpn,spa`) → `add_stereo_audio`
(ensures a 2-channel AAC track exists) → `remove_surround_audio` (drops
6/7/8-channel tracks) → **`check_skip_encode`**

## 5. Encode decision

`check_skip_encode` reads the `skipEncode` flow variable set back in step 2:
- **TRUE** → jump straight to `reorder_streams` (final) — **no FFmpeg run**
- **not TRUE** → `ffmpeg_start`

## 6. FFmpeg build & execute

`ffmpeg_start` → set container **MKV** → set video bitrate to **50% of
input** (fallback 4000k if input bitrate can't be read) → set encoder to
**HEVC, hardware "auto"** (QSV on this box), preset `slow`, quality 20,
hardware decode+encode forced → `ffmpeg_execute`.

## 7. Loopback: reject bloated encodes

`ffmpeg_execute` → **`compare_file_size`** checks the output is between
10%–100% of the original size:
- **In range** → `reorder_streams` (final) — encode accepted
- **Out of range (bloated)** → `set_skip_encode_true` (flip the flag) →
  `delete_bad_cache` (remove the bad output) → `restore_original` (point
  the working file back at the source) → **loops back to `check_mp4`**

That loopback re-runs the entire container/codec/cleanup pipeline (section
3–4) against the *original* file. Since `skipEncode` is now `TRUE`,
`check_skip_encode` will route straight past FFmpeg this second time, so the
file ends up cleaned/remuxed/reordered but not re-encoded — this is the
"bitrate-gate reorder fix" behavior.

## 8. Final safety net & commit

`reorder_streams` (final pass) → **`hard_size_gate`**: a stricter check
(0%–102% of original):
- **Pass** → `replace_original` — the job actually replaces the library file
- **Fail** → `reset_to_original` — dead end, no replacement happens; the
  job ends without touching the original file

---

### Quick reference: nodes with no outgoing edge (deliberate stop points)

- `check_hdr` handle 1 (HDR detected) — HDR files untouched
- `check_file_size_min` handle 2 (<50MB) — tiny files untouched
- `check_video_bitrate` handle 2 (≤1000k, non-HEVC/VP9/AV1) — already-low-bitrate files untouched
- `reset_to_original` — final bloat fallback, original file never replaced
