# File-Native AI Metadata

A standard for embedding semantic context directly inside files — so AI agents understand *what matters* before they read a single byte of content.

---

## The problem

AI agents open files the same way humans do: start from the beginning, read everything, figure it out. That works for a 2-page doc. It breaks for a 90-minute video, a 40-tab spreadsheet, or a 300-page PDF. The agent either processes everything (slow, expensive) or samples naively (misses what matters).

The root issue is that files carry content but not *intent*. There's no standard way to say: "the important part is here, this section is boilerplate, these three timestamps are what the question is actually about."

This project defines that standard — and embeds it inside the file itself.

---

## The idea

Every file gets an optional `__ai_meta__` object, embedded natively in the format's existing metadata layer. No sidecar files. No external registries. The metadata travels with the file, always.

```json
{
  "version": "0.1",
  "purpose": "Human describes what this file is actually for",
  "primary_content": { "start": 0.0, "end": 45.2 },
  "events": [
    {
      "at": 18.7,
      "type": "critical_action",
      "text": "User submits login form",
      "importance": 0.95,
      "causal_link": "leads_to:19.8"
    }
  ],
  "skip_regions": [
    { "start": 0.0, "end": 3.2, "reason": "intro / loading" }
  ],
  "agent_hints": {
    "primary_question": "What caused the error and how was it resolved?",
    "answer_likely_at": [18.7, 19.8, 21.2]
  }
}
```

An agent that reads this before processing the file knows exactly where to look. It doesn't sample 32 frames uniformly — it pulls the 8 frames that actually matter.

---

## Video is the first format

Video is where the cost of not having this is most visible. A 60-second screencast at 30fps has 1,800 frames. State-of-the-art VLMs sample 8–64 of them. The probability of hitting a 0.3-second error state through uniform sampling is ~1%. With metadata, it's 100%.

The `__ai_meta__` object for video lives in the MP4 `udta` box — a container explicitly designed for user-defined metadata. No format modification required. No extra files. The metadata is just there, inside the container, for any agent that knows to look.

### What the demo shows

The [interactive demo](./demo/index.html) runs the same question against the same video twice:

- **Baseline**: uniform 32-frame sampling, standard VLM inference
- **Metadata-guided**: 16 frames selected by `importance_score`, plus semantic bridging text for unsampled gaps

The difference on causal questions ("what caused X", "at what point did Y happen") is not marginal. Agents without metadata confabulate. Agents with it cite exact timestamps.

---

## Format support roadmap

The embedding mechanism differs per format, but the schema is identical.

| Format | Embedding layer | Status |
|--------|----------------|--------|
| MP4 / MOV | `udta` metadata box | ✅ v0.1 |
| PDF | XMP metadata stream | Planned |
| XLSX | Custom XML part (`xl/customXml/`) | Planned |
| DOCX | Custom XML part (`word/customXml/`) | Planned |
| ZIP / archives | Archive comment field | Planned |
| PNG / JPEG | EXIF `UserComment` / XMP | Planned |
| MP3 / audio | ID3v2 `TXXX` frame | Planned |
| HTML | `<meta name="ai-context">` in `<head>` | Planned |

The goal is that any file, in any format, can carry this metadata using only what the format already supports. No new file extensions. No wrapper formats. No external tooling required to read or write it.

---

## Schema reference

### Top-level fields

| Field | Type | Description |
|-------|------|-------------|
| `version` | string | Schema version |
| `purpose` | string | One-sentence description of what this file is for |
| `created_at` | ISO 8601 | When the metadata was authored |
| `primary_content` | object | The region an agent should prioritize |
| `events` | array | Timestamped / positional semantic markers |
| `skip_regions` | array | Ranges the agent can skip entirely |
| `agent_hints` | object | Free-form guidance for the consuming agent |

### Event fields

| Field | Type | Description |
|-------|------|-------------|
| `at` | number | Timestamp (video/audio) or position (page, row, byte offset) |
| `type` | string | Event classification (see types below) |
| `text` | string | Human-readable semantic description |
| `importance` | float 0–1 | How critical this event is to understanding the file |
| `causal_link` | string | `"leads_to:<position>"` or `"caused_by:<position>"` |
| `bounding_box` | object | Spatial region for visual formats (x, y, w, h) |
| `keyframe_flag` | bool | Force-include this position in any sampling strategy |

### Event types

`critical_action` · `state_change` · `error_event` · `ui_interaction` · `navigation` · `form_submit` · `network_request` · `visual_change` · `annotation` · `chapter_start` · `data_anomaly` · `reference`

The type vocabulary is intentionally format-agnostic. `data_anomaly` applies to a spreadsheet cell as much as a video frame. `chapter_start` works for a PDF section as much as a video segment.
