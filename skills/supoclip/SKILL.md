---
name: supoclip
version: 1.0.0
description: Turn long videos (YouTube, Twitch/Kick VODs, direct URLs, uploads) into vertical subtitled clips with SupoClip, including series, topic clips, recaps and trailers, then export them with TikTok/Reels/Shorts presets or publish them. Use when the user mentions SupoClip, or wants clips made through the SupoClip API or MCP server instead of OpenShorts.
homepage: https://github.com/FujiwaraChoki/supoclip
metadata:
  openclaw:
    emoji: "✂️"
    primaryEnv: SUPOCLIP_API_KEY
  hermes:
    category: media
    tags: [video, clips, shorts, social-media, supoclip]
---

# SupoClip: clip a long video

SupoClip (AGPL-3.0, github.com/FujiwaraChoki/supoclip) finds the most clip-worthy
moments of a long video, scores them (hook, engagement, value, shareability) and
renders them as 9:16 clips with face-centred crop, word-synced captions and an
optional hook title and B-roll. A task takes minutes: always submit, then wait.

## Connect

Prefer MCP when the host supports it; fall back to REST.

- **MCP (hosted)**: `https://mcp.supoclip.com/mcp`, header
  `Authorization: Bearer sk_...`. Claude Code:
  `claude mcp add --transport http supoclip https://mcp.supoclip.com/mcp --header "Authorization: Bearer $SUPOCLIP_API_KEY"`
- **MCP (local stdio, saves MP4s on this machine)**:
  `claude mcp add supoclip -e SUPOCLIP_API_KEY=sk_... -- uvx --from "git+https://github.com/FujiwaraChoki/supoclip#subdirectory=mcp" supoclip-mcp`
  Add `-e SUPOCLIP_API_URL=http://localhost:8000` for a self-hosted backend.
- **REST**: `https://api.supoclip.com` (self-host: `http://localhost:8000`),
  header `Authorization: Bearer sk_...` or `x-api-key: sk_...`. Payloads and
  error shapes are in `reference.md`; read it before the first HTTP call.

Keys (`sk_...`) are created at supoclip.com → Settings → API Keys and need a
verified email. **Creating tasks on the hosted service needs a paid plan**; on
a free account discovery calls work and task creation returns 402.

Keys are SupoClip keys, not OpenShorts keys: an `osk_` key will be rejected and
vice versa. Never send one service's key to the other.

## The core loop

1. `supoclip_billing_summary` (`GET /tasks/billing/summary`) before a large job.
2. Anything other than plain highlights, or any `instructions`: call
   `supoclip_list_workflows` (`GET /tasks/workflows`) first. `enabled: false`
   means only highlights without instructions.
3. Submit: `supoclip_create_clip_task` / `POST /tasks/` with
   `{"source": {"url": "..."}}` plus the options below. Returns `task_id`.
4. Wait: `supoclip_wait_for_task`, or poll `GET /tasks/{id}` every 15-30 s
   until `status` is `completed`, `error` or `cancelled`.
5. Results: `supoclip_list_clips` (`GET /tasks/{id}/clips`), each with
   `start_time`, `end_time`, `virality_score`. Download with
   `supoclip_download_clip` or export with `supoclip_export_clip`
   (`preset`: `tiktok`, `reels`, `shorts`). On the **hosted** MCP server those
   two save files on the server, not here: use the local stdio server or REST.

## Choosing options

- **workflow**: `highlights` (default, best standalone clips), `series`
  (Part 1..N of one story, `part_count` auto or 2-10, source ≥ 2 min),
  `topic` (every strong moment about `topic`, 2-120 chars, source ≥ 1 min),
  `recap` (one video of the main points, ≥ 3 min), `trailer` (one teaser that
  never reveals the payoff, ≥ 3 min). `topic` with another workflow, or
  `part_count` outside a series, is an error, not ignored.
- **clip_length**: `auto` (15-60 s per clip), `short`, `medium`, `long`. For
  series it is per part; for recap and trailer it is the **total** length.
  Always set it explicitly: the API does not read the account's saved default.
- **instructions**: up to 500 chars of editorial guidance for any workflow
  ("skip the sponsor read"). Use the user's words; do not invent them.
- **output_format**: `vertical` (crop, default), `vertical_fit` (whole frame on
  9:16), `vertical_pan`, `vertical_split`, `original`.
- **Captions**: `add_subtitles`, `caption_template` (list them with
  `supoclip_list_caption_templates`: default, hormozi, mrbeast, ...), optional
  `font_options` `{font_family, font_size 12-72, font_color "#RRGGBB"}`.
- **include_broll**: only if `supoclip_broll_status` says configured.
- `processing_mode` `fast` | `balanced` | `quality`; `cut_long_pauses` and
  `remove_filler_words` default on.

## Failures that need a decision, not a retry

- **402 / `SUBSCRIPTION_REQUIRED`**: no plan or no minutes. Report the upgrade
  link and stop.
- **403 `WORKFLOW_NOT_AVAILABLE`**: fall back to highlights only if the user agrees.
- **422** codes (`TOPIC_REQUIRED`, `SOURCE_TOO_SHORT_FOR_*`, `TOO_MANY_PARTS`
  with `max_parts`, ...) are raised before anything is charged: fix the argument.
- **Task ends `error` with an `error_code`**: minutes are refunded. Follow the
  table in `reference.md`: `topic_not_found` offers `suggested_topics`;
  `no_story` → recap; `trailer_unbuildable` → recap; `analysis_incomplete`,
  `plan_failed`, `render_incomplete` → `supoclip_resume_task` once.
- **429**: wait, do not resubmit.

## Rules

- Only clip videos the user has rights to.
- Publishing (`POST /social/posts`) is public: confirm account, caption and
  schedule with the user first. Connecting accounts happens in the web app.
- `supoclip_delete_task` is irreversible: confirm before calling it.
- When the user just wants clips and has no preference, OpenShorts (the
  `openshorts` skill) does the same job; use SupoClip when they ask for it or
  already have a SupoClip key.
