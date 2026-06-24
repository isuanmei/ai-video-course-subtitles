---
name: ai-video-course-subtitles
description: Add, correct, and burn subtitles for AI影像实战课程 finished videos. Use when the user asks to 给AI影像实战课/成片/无字幕视频上字幕, revise burned-in subtitle timing/text, match the first-lesson subtitle style, avoid duplicating existing on-screen subtitles, or preserve course-specific terms such as 即梦, 可灵, vidu, Seedance, GPT Image, 首尾帧, 图生视频.
---

# AI影像课程字幕

Use this skill for finished AI影像实战课程 videos that need burned-in Chinese subtitles or subtitle corrections.

## Core Rules

- Preserve the original video; write a new `_字幕版.mp4` unless the user is asking to update the current generated subtitle version.
- Match the first-lesson subtitle style: bottom centered, one line, bold white Chinese text, black outline, no sentence-final punctuation.
- Do not add subtitles during the intro/title-card portion. Start subtitles at the first audible正文口播 and keep the subtitle timecode aligned with the voice, not with the visual transition alone.
- Keep one subtitle line as one short sentence or spoken meaning unit in normal cases. Split long sentences at natural pauses; do not cram long text into one line.
- Do not duplicate video parts that already have subtitles,歌词,案例视频字幕, or dense on-screen text meant to be read directly.
- Treat transcription as raw material. Always run a course-specific correction pass before burning.
- Keep names exact: `即梦`, `可灵`, `vidu`, `Seedance`, `GPT Image`, `首尾帧`, `图生视频`, `文生视频`, `一镜到底`, `滑板女孩`.
- When a skip interval only overlaps part of a transcript segment, crop the time range rather than dropping the whole segment.

## Workflow

1. Locate the source video in `成片/` and inspect metadata with `ffprobe`.
2. Create a per-video subtitle workspace under `成片/`.
3. Extract audio and transcribe. Prefer local transcription when available; ask before uploading audio to an external service.
4. Build a cleanup script in the workspace that loads the transcript JSON, applies overrides/replacements, crops skip intervals, splits long lines, and writes `.srt` plus `.ass`.
5. Inspect the generated SRT around known fragile spots: intro, tool names, examples, existing-subtitle sections, and ending.
6. Burn subtitles with `ffmpeg`/`libass` into `_字幕版.mp4`.
7. Verify with text scans and extracted frames before finalizing.

For the exact reusable implementation pattern, read [references/workflow.md](references/workflow.md).

## Burn Style

Use ASS subtitles for predictable 4K output:

```text
PlayResX: 1920
PlayResY: 1080
Fontname: Hiragino Sans GB
Fontsize: 54
PrimaryColour: white
OutlineColour: black
Bold: true
Outline: 2.4
Shadow: 0.8
Alignment: bottom center
MarginV: 58
```

Use `-pix_fmt yuv420p`, keep audio copied, and add `-movflags +faststart`.

## Validation

Before reporting completion:

- Confirm first subtitles start after the intro, at the first正文 voice timecode, and stay aligned with the audio.
- Run `rg` for known wrong forms such as `吉梦|极梦|吉孟|寂寞|克林|可林|微杜|微度|花瓣`.
- Extract and visually inspect frames at the intro boundary, corrected term locations, existing-subtitle skip sections, and the ending.
- Confirm output dimensions, frame rate, and duration with `ffprobe`.
- State the output path, files changed, and checks performed.
