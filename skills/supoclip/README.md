# SupoClip as an Agent Skill

`SKILL.md` plus `reference.md` teach an agent to drive
[SupoClip](https://github.com/FujiwaraChoki/supoclip), an open-source (AGPL-3.0)
clip generator, through its MCP server or REST API: which workflow to pick,
what `clip_length` means for each, and which errors need the user instead of a
retry. It sits next to `skills/openshorts/` so an agent can use either service.

This folder only contains instructions; no SupoClip code is vendored here.

## Setup

Create a key (`sk_...`) at supoclip.com → Settings → API Keys and expose it as
`SUPOCLIP_API_KEY`. Creating clips on the hosted service needs a paid plan.
For a self-hosted SupoClip set `SUPOCLIP_API_URL` to its backend.

Add the MCP server so the agent gets typed tools:

```bash
claude mcp add --transport http supoclip https://mcp.supoclip.com/mcp \
  --header "Authorization: Bearer $SUPOCLIP_API_KEY"
```

or run it locally (clips download to this machine):

```bash
claude mcp add supoclip -e SUPOCLIP_API_KEY=sk_... \
  -- uvx --from "git+https://github.com/FujiwaraChoki/supoclip#subdirectory=mcp" supoclip-mcp
```

## Install

```bash
cp -r skills/supoclip ~/.claude/skills/
```

Any other Agent Skills host: drop the folder wherever it reads skills from.
