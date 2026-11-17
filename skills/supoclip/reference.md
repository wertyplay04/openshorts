# SupoClip HTTP reference

Base URL: `https://api.supoclip.com` (self-hosted: your backend, usually
`http://localhost:8000`, interactive docs at `/docs`). Auth on `/tasks/*`,
`/fonts` and `/upload`: `Authorization: Bearer sk_...` or `x-api-key: sk_...`.
`/caption-templates`, `/transitions` and `/broll/status` need no auth.

Self-hosted backends also accept signed headers (`SUPOCLIP_USER_ID` +
`BACKEND_AUTH_SECRET`) or, with `ALLOW_UNSIGNED_BACKEND_AUTH=true`, just a user
id. Prefer an API key.

The MCP server (`mcp/` in the SupoClip repo, hosted at
`https://mcp.supoclip.com/mcp`) wraps these same endpoints.

## Discovery

| Call | Returns |
|---|---|
| `GET /tasks/billing/summary` | `plan`, `subscription_status`, `usage_minutes`, `minute_limit`, ... |
| `GET /tasks/workflows` | `enabled`, `workflows[].{id, available, min_source_seconds, lengths, part_count}`, `topic_max_chars` (120), `instructions_max_chars` (500) |
| `GET /caption-templates` | templates with `font_family`, `font_size`, `font_color`, animation |
| `GET /fonts` | fonts usable as `font_options.font_family` |
| `GET /transitions` | `{transitions: [{name, display_name}]}` |
| `GET /broll/status` | `{configured: bool}` |

## Create a task

`POST /tasks/`

```json
{
  "source": {"url": "https://www.youtube.com/watch?v=VIDEO_ID", "title": "optional"},
  "processing_mode": "fast",
  "clip_length": "auto",
  "output_format": "vertical",
  "add_subtitles": true,
  "caption_template": "default",
  "include_broll": false,
  "font_options": {"font_family": "...", "font_size": 32, "font_color": "#FFFFFF"},
  "cut_long_pauses": true,
  "remove_filler_words": true,
  "workflow": "highlights",
  "workflow_options": {"topic": "pricing", "part_count": "auto"},
  "instructions": "Skip the sponsor read"
}
```

Only `source` is required. Response: `{"task_id", "job_id", "message"}`.
To clip a local file, `POST /upload` (multipart) first and use what it returns
as the source.

`clip_length` meaning per workflow:

| workflow | auto | short | medium | long |
|---|---|---|---|---|
| highlights, topic (per clip) | 15-60 s | 15-30 | 30-60 | 60-120 |
| series (per part) | 45-150 | 30-60 | 45-90 | 90-175 |
| recap (total) | scales, 45-120 | ~45 | ~75 | ~120 |
| trailer (total) | ~30 | ~20 | ~40 | ~60 |

Rejections before anything is created or charged (`detail: {code, message}`):

| Status | code |
|---|---|
| 422 | `INVALID_WORKFLOW`, `INVALID_WORKFLOW_OPTIONS`, `TOPIC_REQUIRED`, `TOPIC_TOO_LONG`, `INVALID_PART_COUNT`, `INSTRUCTIONS_TOO_LONG` |
| 422 | `SOURCE_TOO_SHORT_FOR_SERIES` / `_RECAP` / `_TRAILER` / `_TOPIC` (with `min_source_seconds`), `TOO_MANY_PARTS` (with `max_parts`) |
| 403 | `WORKFLOW_NOT_AVAILABLE` |
| 402 | `SUBSCRIPTION_REQUIRED` |
| 429 | rate limited |

## Track a task

- `GET /tasks/{id}`: `status` (`queued`, `processing`, `completed`, `error`,
  `cancelled`), `progress`, `progress_message`, `error_code`, `clips`,
  `workflow`, `workflow_plan`, `source_duration_seconds`.
- `GET /tasks/{id}/progress`: Server-Sent Events, including `clip_ready`.
- `GET /tasks/`: list. `POST /tasks/{id}/cancel`, `POST /tasks/{id}/resume`,
  `DELETE /tasks/{id}`.

Workflow failures end the task `error` with no clips and refunded minutes:

| error_code | Next step |
|---|---|
| `topic_not_found` | Retry with one of `workflow_plan.error.details.suggested_topics`, or highlights |
| `source_too_short` | Highlights |
| `no_story` | Recap |
| `recap_unsupported` | Highlights |
| `trailer_unbuildable` | Recap |
| `analysis_incomplete`, `plan_failed`, `render_incomplete` | Resume |
| `transcript_untimed` | Upload the file instead of the URL |

## Clips

- `GET /tasks/{id}/clips` → `{task_id, clips: [{id, filename, start_time,
  end_time, virality_score, workflow_role, part_number, workflow_meta}],
  total_clips}`
- `GET /tasks/{id}/clips/{clip_id}/file`: the MP4.
- `PATCH /tasks/{id}/clips/{clip_id}` edit, `POST .../split`,
  `POST /tasks/{id}/clips/merge`, `PATCH .../captions`, `POST .../regenerate`,
  `DELETE .../{clip_id}`.
- `POST /tasks/{id}/settings`: task-wide font / caption template.

## Export with a platform preset

- Async (preferred): `POST /tasks/{id}/clips/{clip_id}/exports` with
  `{"preset": "tiktok" | "reels" | "shorts"}` → export id; poll
  `GET .../exports/{export_id}` (`queued`, `running`, `succeeded`, `failed`,
  `progress`, `download_url`); fetch `GET .../exports/{export_id}/file`.
- Sync: `GET /tasks/{id}/clips/{clip_id}/export?preset=tiktok` blocks until
  ffmpeg finishes.

## Share and publish

- `POST /tasks/{id}/share` / `DELETE` → read-only link; public at
  `GET /tasks/shared/{share_token}`.
- `GET /social/connections`: connected YouTube / TikTok / Instagram accounts
  (connect them in the web app, which needs a browser session).
- `POST /social/posts` with `{task_id, clip_id, social_account_id, title?,
  caption?, hashtags?, privacy_level?, scheduled_for?}`. No `scheduled_for`
  posts now. TikTok `privacy_level` defaults to `private`.
- `GET /social/posts`, `POST /social/posts/{id}/cancel`, `.../retry`.
