# AI影像课程字幕 Workflow Reference

## Directory Pattern

For a source such as:

```text
成片/AI影像实战第五课无字幕v2.m4v
```

Create:

```text
成片/第五课字幕工作区/
├── whisper_full/
│   └── <video>.json
├── <video>_clean_segments.json
├── <video>_clean.srt
├── <video>_no_double.ass
├── build_subtitles.py
└── checks/
```

Output:

```text
成片/<video>_字幕版.mp4
```

## Cleanup Script Structure

Use a deterministic `build_subtitles.py` in the workspace. Core pieces:

```python
SKIP_INTERVALS = []  # add intro/case/lyrics/existing-subtitle ranges after checking the video

REPLACEMENTS = [
    ("吉孟", "即梦"), ("吉梦", "即梦"), ("极梦", "即梦"), ("寂寞", "即梦"),
    ("克林", "可灵"), ("可林", "可灵"), ("可憐", "可灵"), ("可怜", "可灵"),
    ("微杜", "vidu"), ("微度", "vidu"),
    ("花瓣女孩", "滑板女孩"),
    ("她花瓣的这个画面", "她滑板的这个画面"),
    ("手尾针", "首尾帧"), ("守卫针", "首尾帧"), ("尾针", "尾帧"),
    ("一径到底", "一镜到底"),
    ("图身视频", "图生视频"), ("图伸视频", "图生视频"),
    ("文伸视频", "文生视频"), ("文身视频", "文生视频"),
    ("深沉", "生成"), ("生辰", "生成"),
    ("樱画同出", "音画同出"), ("生效和它的动效", "音效和它的动效"),
]
```

Keep user-confirmed corrections in the replacements list so they survive regenerations.

## Skip/Crop Logic

Do not use a simple overlap test that drops a whole transcript segment. If a transcript segment overlaps the intro or a no-subtitle section, crop only the overlapped time:

```python
def visible_ranges(start, end):
    ranges = [(start, end)]
    for skip_start, skip_end in SKIP_INTERVALS:
        next_ranges = []
        for range_start, range_end in ranges:
            if range_end <= skip_start or range_start >= skip_end:
                next_ranges.append((range_start, range_end))
                continue
            if range_start < skip_start:
                next_ranges.append((range_start, skip_start))
            if range_end > skip_end:
                next_ranges.append((skip_end, range_end))
        ranges = next_ranges
    return [(a, b) for a, b in ranges if b - a >= 0.1]
```

For 第五课 only, the intro skip interval was `0.0-3.13`, not `0.0-11.0`; the first visible subtitle is:

```text
00:00:03,130 --> 00:00:04,907
各位同学大家好
```

For a new lesson, determine the intro end by listening and checking frames. The rule is: no subtitles during the片头/title-card-only section; the first subtitle begins when正文口播 sound begins and must match that audio timecode.

The fifth-lesson case/lyrics segment was skipped with an approximate interval `1600.0-1810.0`. Re-detect such intervals for each new lesson by sampling frames; do not blindly copy this range.

## Text Rules

- One screen, one line.
- A line is normally one complete short sentence or spoken meaning unit.
- Split long sentences at natural pauses instead of only by fixed character count.
- Keep lines short enough to read comfortably; target concise lines and avoid exceeding about 24 Chinese characters unless the wording is already very compact.
- Remove sentence-final punctuation.
- Keep useful inner spaces for mixed Chinese/English phrases such as `GPT Image`, `Seedance 2.0`, `vidu`.
- Use manual overrides for long or important segments whose transcription is poor.

## Burn Command Pattern

```bash
/usr/bin/env FC_CACHEDIR=/private/tmp/fontconfig-cache-codex \
ffmpeg -y -i "成片/source.m4v" \
  -vf "ass=成片/workdir/source_no_double.ass" \
  -c:v libx264 -preset veryfast -crf 18 -pix_fmt yuv420p \
  -c:a copy -movflags +faststart \
  "成片/source_字幕版.mp4"
```

If hardware encoding fails or changes quality unexpectedly, use `libx264`.

## Review Checklist

Text scans:

```bash
rg -n "吉梦|极梦|吉孟|寂寞|克林|可林|可怜|微杜|微度|花瓣|生辰|深沉|手尾针|守卫针|一径到底" clean.srt
rg -n "即梦|可灵|vidu|滑板女孩" clean.srt
```

Frame checks:

- Before the first subtitle: no口播字幕.
- At first subtitle timestamp: `各位同学大家好`.
- Tool-name frames: `即梦`, `可灵`, `vidu`.
- User-corrected phrases: for 第五课, `让滑板女孩模仿` and `她滑板的这个画面`.
- Skip intervals: no duplicate口播字幕 on existing-subtitle/case/lyrics content.
- Ending: final subtitle is present and readable.

Metadata:

```bash
ffprobe -v error -select_streams v:0 \
  -show_entries stream=width,height,r_frame_rate,duration \
  -of default=nw=1 "成片/source_字幕版.mp4"
```

Report output path, subtitle workspace path, files changed, and whether any checks could not be run.
