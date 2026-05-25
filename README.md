# Resurvey skill for Claude Code

Build interactive surveys through conversation with Claude Code. The skill drives a phased authoring loop (frame → outline → compose → pressure-test → publish) and writes two files: `survey.config.ts` and `src/survey.tsx`. The runtime, dev-server, event pipeline, and dashboard at [resurvey.teleios.au](https://resurvey.teleios.au) are already generic.

## Install

```
/plugin marketplace add resurvey-dev/resurvey-skill
/plugin install resurvey@resurvey-dev
```

Then:

```
build a survey for measuring NPS on our paid users
```

Claude will scaffold a project, ask the framing questions, compose the survey, run a preflight, and hand you a shareable link.

## What the skill writes

- `survey.config.ts` — survey metadata, preview defaults, notes/sources.
- `src/survey.tsx` — the React component composing question blocks from `@resurvey/survey-runtime`.
- `CRITIQUE.md` — self-critique of bias, drop-off risks, anonymity claims.

## What the skill doesn't touch

- `package.json`, `vite.config.ts`, `tsconfig.json`, `index.html`, `src/main.tsx`, or anything in `node_modules/`.
- The runtime itself (`@resurvey/survey-runtime`). Engine improvements happen in the runtime repo and reach existing surveys on the next `resurvey deploy`.

## Capabilities

- Standard question blocks: `<SingleChoice>`, `<MultiChoice>`, `<Likert>`, `<NPS>`, `<FreeText>`.
- Distribution types: `public`, `password`, `token` (per-respondent resume), `panel` (with required params).
- Event pipeline: client-generated event ids for idempotent retries, beacon-on-unload for tab-close survival, automatic answer-revision tracking, sticky `completed_at`.
- Real-time response tail in Studio (`resurvey dev`), JSONL export (`resurvey responses pull`).

## Not for

- Drafting interview guides or qualitative protocols where you'd talk to people one-on-one.
- Static questionnaires that don't get distributed (use a Google Form).
- Surveys requiring HIPAA/clinical compliance — this is a research-grade tool, not a regulated instrument host.

## Self-hosting

The skill talks to `https://resurvey.teleios.au` by default. To point at your own backend, set the API base URL when running the CLI:

```
resurvey login --api-base-url https://your-host.example.com
```

The full backend lives at [github.com/resurvey-dev/resurvey-service](https://github.com/resurvey-dev/resurvey-service).
