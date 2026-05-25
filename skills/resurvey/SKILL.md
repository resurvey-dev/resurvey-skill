---
name: resurvey
description: Use this skill when the user wants to build, edit, or publish a Resurvey survey — a React+Vite interactive survey that collects responses through the resurvey.teleios.au backend, with public/password/token/panel distribution. Triggers include "build a survey for X", "make a resurvey to ask Y", "set up an NPS / CBC / pulse / feedback survey", "turn this question into a survey I can share", or any request that ends with the user wanting a shareable link to a survey their respondents fill in themselves. Not for: drafting interview guides, static questionnaires that don't get distributed, or qualitative-only research where you'd interview people one-on-one.
---

# Resurvey skill

## What you'll do

When the user wants to learn something from a group of respondents:

1. **Scaffold the project** with `resurvey init` — a React + Vite template with `@resurvey/survey-runtime` already wired.
2. **Author `survey.config.ts` and `src/survey.tsx`** — the two files that carry the survey. The runtime blocks, dev-server, event submission, and dashboard are already generic.
3. **Pre-flight** — read every question out loud, run the gates, write `CRITIQUE.md`.
4. **Run + publish** — `resurvey dev` for local preview, `resurvey deploy` + `resurvey distribute create` for the share link.

Don't touch `package.json`, `vite.config.ts`, `tsconfig.json`, `index.html`, `src/main.tsx`, or anything in `node_modules/`. If the user asks for a capability the runtime doesn't have (e.g. a new block type), do it in the **runtime repo** (`@resurvey/survey-runtime`), not in their survey.

## Step 1 — Scaffold

Derive a short slug from what the user is trying to learn (`saas-q3-nps`, `pricing-cbc-2026`, `oncall-pulse-mar`):

```sh
resurvey init --template <id> --dir ./<slug>
cd ./<slug>
```

Pick the template that best matches the survey shape:

| Template | When |
|---|---|
| `cx`          | NPS or short customer-feedback flows (1–3 questions, one open-ended) |
| `pricing-cbc` | Conjoint / choice-task pricing experiments |
| `basic`       | Multi-step surveys with mixed question types (Likert, single-choice, free text) |
| `blank`       | Anything else — minimal shell intended for agent-generated flows |

If `<slug>/survey.config.ts` already exists, **edit in place — do not re-scaffold**.

If `resurvey` isn't installed: `npm install -g resurvey` once, then proceed. If npm install fails (private alpha), fall back to:
```sh
git clone --depth 1 https://github.com/resurvey-dev/resurvey-cli.git ../.resurvey-cli
npx --prefix ../.resurvey-cli resurvey init --template <id> --dir ./<slug>
```

## Step 2 — Understand the page so you can author for it

The respondent sees a single React SPA served from `resurvey.teleios.au/d/<distId>` (with a brief landing for password/token distributions). Each question is rendered by a block from `@resurvey/survey-runtime`: `<SingleChoice>`, `<MultiChoice>`, `<Likert>`, `<MatrixLikert>`, `<NPS>`, `<FreeText>`, `<CBCChoice>`, plus layout helpers (`<Layout>`, `<Card>`, `<Progress>`, `<Button>`). The skill composes these — it doesn't roll its own form controls.

Every interaction emits an event (`trackStart`, `trackAnswer`, `trackComplete`); the event stream is the data the author later analyses. **The runtime tracks answer revisions automatically** — if the respondent changes their answer to Q3 from "No" to "Yes", both are in the event log with `revision: 1` and `revision: 2`. Author code shouldn't reimplement this.

Distribution types — picked at publish time, but the survey design has to support them:

| Type | What it does | When |
|---|---|---|
| `public`   | One URL, anonymous sessions, no resume | General-public surveys, no PII, you want broad reach |
| `password` | One URL + one shared password | Internal pulse, semi-private audience |
| `token`    | Pre-issued URLs (`/d/X?t=Y`), one per respondent, sessions resume on return | Panel studies, identified respondents who may pause and come back |
| `panel`    | URL with required query params (`?pid=...`), each param value mints a session | Vendor panels, identified by external id |

The survey component must not assume the respondent comes back (drop `useEffect`-based resume logic) unless the distribution type is `token` or `panel`. The runtime exposes `ctx.distribution.type` for branching.

## Voice & writing rules

**Read this before writing any question, answer, or label.** A respondent reads each question once. They don't re-read. They don't translate. Every word that asks them to think about *what you mean* is a word that costs you data quality — or worse, makes them drop off mid-survey.

### Who answers this

A normal person on their phone, in five idle minutes, who chose to give you their attention because the topic is interesting OR they got something in exchange. They:

- **Know:** their own experience. What they bought, how they felt, what they tried, what didn't work.
- **Don't know:** your jargon. Your product vocabulary. Your industry acronyms. Anything that requires you to be in the room explaining.

Write like you're asking a friend across the table. Not a customer-service form. Not a market-research questionnaire. Not a clinical instrument.

### Question patterns to avoid, and what to use instead

| ✗ Don't write | ✓ Write |
|---|---|
| How satisfied are you with our product? (Likert 1–5) | The labels matter — what's a "3"? Anchor every option: "Very dissatisfied / Dissatisfied / Neither / Satisfied / Very satisfied" |
| Do you find our app easy to use AND useful? | One thing per question. Ask easy-to-use OR useful, not both. |
| You wouldn't NOT recommend us, would you? | Direct: "Would you recommend us?" No double negatives. |
| Rate our customer support (1 to 10) | Anchor the scale: "How well did support resolve your issue? 1 = didn't resolve, 10 = fully resolved" |
| How often do you use the platform? | Specific intervals: "In the last 7 days, how many days did you open the app?" |
| Are you satisfied? Why? | Two questions back-to-back. Don't bury the why on the same screen. |
| What features would you want? | Open-ended fishing. Surface concrete options + an "Other" + a follow-up box. |
| How much would you pay? (free text) | Anchor with options. "Which feels closest to fair: $5 / $15 / $30 / $50+ / depends." |
| Do you agree that our pricing is fair? | Agree-disagree on a value statement is meaningless. Ask what they *would* do: "Would you renew at the current price?" |
| Compared to industry leaders, how does our product perform? | Who's the industry leader in their head? Drop the comparison or name it. |
| Please rank these 12 options. | Nobody ranks 12 things on a phone. Cap at 5 with drag-rank, or ask top-3 selection. |
| Mandatory open-ended after every question | One open-ended at the end is enough. Mandatory comments tank completion rate. |

### Naming patterns

**Question text** is one sentence the respondent could repeat to themselves before answering. 5–20 words. No "the following" preambles.

| ✗ | ✓ |
|---|---|
| Please indicate the extent to which you agree with the following statement: "I find the platform intuitive." | How easy was the platform to figure out the first time? |
| To what degree does our solution meet your operational requirements? | Does it do what you need it to do? |
| Which of the following best describes your engagement with the service? | When did you last use it? |

**Answer choices** are short and parallel. Each option is the same grammatical shape; one option doesn't read in a different voice than the others.

| ✗ Unparallel | ✓ Parallel |
|---|---|
| Daily / A few times a week / I rarely log in / Almost never  | Every day / A few times a week / About once a week / Less than once a week |
| Yes / Maybe / Why would you ask that?  | Yes / No / Not sure |
| Strongly agree / Agree / Neutral / Disagree / I don't really know | Strongly agree / Agree / Neither / Disagree / Strongly disagree |

**Open-ended prompts** invite a specific answer. Not "any other comments." Not "feedback?".

| ✗ | ✓ |
|---|---|
| Any other comments? | What's the one thing that would make you recommend this to a friend? |
| Feedback? | What's the most annoying thing about your current workflow? |
| Why did you give that score? | You said "7" — what would have made it a "10"? |

### Stress-test before declaring done

Read every question and every answer out loud. Ask:

1. **Could the respondent answer this in one breath?** If they hesitate to parse it, drop a clause. Aim for <20 words on question text.
2. **Is there exactly one thing being asked?** Two clauses joined by AND/OR almost always means split into two questions.
3. **Does each answer choice have an obvious meaning?** "Sometimes" is not an answer choice. "About once a month" is.
4. **Would the answer change the decision?** If yes-or-no gives you the same next step, cut the question.
5. **What's the longest path through the survey?** Count screens. If >10, justify the length out loud or cut.

If any question fails, rewrite before moving on. "Almost-plain" reads worse than fully technical — the inconsistency makes the respondent doubt the rest.

## Step 3 — Author the survey

Work through these in order. Most surveys fail at A–C, not at the question text.

### A. Frame what you actually want to learn (ask once)

Before writing any question, four facts must be clear. If unclear from the brief, **ask all of them in a single combined message**:

- **The decision this informs.** "What will you do differently if the answer is X vs. Y?" If the answer is "nothing", don't run the survey — save everyone's time.
- **The audience.** Who answers, how you reach them, how many you need (and how many you'll realistically get).
- **The shape of the answer.** A number? A list of priorities? A free-text theme to cluster? Different shapes warrant very different surveys.
- **Sensitivity.** Is anything you're asking emotionally loaded, embarrassing, or personally identifying? Anonymity claims must match reality — if the distribution is `public`, you can't promise tracking-free; if it's `token`, you can't promise anonymous-from-the-author.

Write the framing into the top of `survey.config.ts` as `description` and `meta` fields. The author should be able to read it back and say "that's what I'm trying to learn."

### B. Pick the audience and distribution type

Two coupled choices:

- **Sample size target (N).** Order-of-magnitude is enough at design time. <30 = qualitative pulse; 30–200 = quantitative-ish; >200 = statistically defensible for headline numbers.
- **Distribution type.** See the table in Step 2. Default to `public` for breadth, `token` for panel studies, `password` for internal-only.

If the user names a sample size that doesn't match the recruitment plan (e.g., "I want 500 responses but I'll just share it in my Slack DMs"), say so up front. Better to renegotiate now than discover at N=8 that the link was never going to reach 500 people.

### C. Outline the survey before composing

Draft the question list as a flat outline first:

```
1. screener: does this person belong in the sample?
2. main: the question that actually matters
3. driver: one or two why-questions to interpret #2
4. demographics (only the ones that will be cross-tabbed)
5. open-ended: one final box
```

Constraints:

- **Default cap: 10 questions, 5 minutes.** Every question beyond that needs to earn its place. Long surveys lose mid-survey respondents — and those who finish are no longer representative.
- **Branches are expensive.** Each branch doubles QA surface. Use them when paths diverge meaningfully (screener filters out off-topic users); avoid them for "if you said 8+, do this extra question" sprawl.
- **Easy first, sensitive last.** Demographics and sensitive items go after the substantive questions. The respondent has already invested time; quitting at Q9 is rare.
- **Randomize where order biases the answer.** Single-choice option lists of >3 items: shuffle them on a per-session seed (use `ctx.assignment.seed`). The runtime provides deterministic seeds so test runs are reproducible.

### D. Compose with runtime blocks

Use the blocks from `@resurvey/survey-runtime` for every standard question type. The block library is the canonical UI — recipients have seen it before, accessibility is handled, mobile sizing works, event submission is automatic.

| Block | When |
|---|---|
| `<SingleChoice options={...} />` | One answer from a small list (<= 7) |
| `<MultiChoice options={...} maxSelections={N} />` | Multiple answers from a small list, with a max |
| `<Likert anchors={["...", "..."]} points={5} />` | Agreement / frequency / quality scales, with anchored endpoints |
| `<NPS />` | The exact 0–10 recommend question with the standard labels |
| `<FreeText placeholder="..." maxLength={N} />` | Open-ended; default `maxLength: 500` so respondents don't write essays |
| `<MatrixLikert statements={[...]} labels={{min, max}} />` | A shared scale across several short statements ("5 statements about on-call") — one screen, one set of anchors |
| `<CBCChoice task attributes value onChange />` + `generateCBCDesigns({ attributes, numTasks, seed })` | Conjoint / pricing-CBC choice tasks. Generate the design once from `ctx.sessionId`, render one task per screen, store the chosen `alternativeIndex` per task |
| `<Layout>` `<Card>` `<Progress>` `<Button>` | Composition — wrap a question in a card, render the progress bar, primary action |

**Submit pattern.** Every block fires `trackAnswer({ questionId, value })` on change. The runtime handles revisions, batching, retries, beacon-on-unload. **Don't** wire your own `fetch` to the events endpoint — you'll skip the queue, lose offline submissions, and break dedup.

Only drop to raw React when the runtime genuinely doesn't cover the interaction (CBC choice tasks, drag-rank, custom visual selectors). When you do, still call `submitSafe({ type: "answer", questionId, value })` so the answer rides the same pipeline.

### E. Source any benchmark or instrument citations

If the survey uses an established instrument (PSS, SUS, validated NPS prompt, established CBC design), keep the wording exactly as published and cite the source in `survey.config.ts.notes`. Don't paraphrase a validated instrument — the validation no longer applies.

For sample-size or detection-power claims ("we'll be able to detect a 5pp shift in NPS"), web-search a current source and cite it. If results are thin, prefer the conservative estimate and note the uncertainty.

### F. Critique each question

For every question, ask:

1. **Does the answer change a downstream decision?** If no, cut.
2. **Is there a simpler proxy that's already in the user's data?** If yes, don't ask — use the existing data.
3. **What's the likely bias?** Acquiescence (yea-saying), social desirability (looking good to the surveyor), recency, anchoring. Name the strongest bias for each question in a code comment.
4. **What happens to the analysis if 30% skip this question?** Optional vs. required is a meaningful choice — required questions raise drop-off; optional questions break cross-tabs.

### G. Self-critique pass — `CRITIQUE.md`

Before declaring done, write `CRITIQUE.md` next to `survey.config.ts` answering in writing:

1. *What's the most likely way this survey produces a misleading answer?* (Question wording bias, sample bias, drop-off pattern, social desirability — name the strongest.)
2. *What's the question this survey is secretly asking but pretending not to?* (e.g. NPS is framed as "would you recommend us" but is really about whether the answer is high enough to email to investors.)
3. *Who would you not show this survey to, and why?*

These are the questions a methodologist will ask. Better to find them yourself first.

### H. Pre-flight checklist

Hard gates before declaring done:

- `survey.config.ts.meta.description` written, frames the decision the survey will inform.
- Question count ≤ 10 (or explicit justification in `CRITIQUE.md` for going longer).
- Every required question is answerable in one breath (the stress test above).
- Every question type is a runtime block, OR the deviation is justified in a comment.
- Order: screener → main → drivers → sensitive → open-ended.
- `trackComplete()` fires at the end of the happy path.
- `npm run build` succeeds with no console errors.
- `resurvey dev` preview runs through the survey end-to-end including the complete step.
- `CRITIQUE.md` exists.

If a gate fails, fix it before telling the user it's done.

## Step 4 — Run + publish

### Local preview

```sh
resurvey dev
```

Opens Studio (preview + event tail) at `http://localhost:5173` and the survey at `http://localhost:5174`. Hot reload on every save. Events stream into Studio's tail; click "Reset session" to start over.

### Publish

```sh
resurvey deploy
resurvey distribute create --type public
```

`deploy` builds the survey, uploads the bundle, finalizes the release. `distribute create` mints a shareable URL. For password/token/panel:

```sh
resurvey distribute create --type password --password "..."
resurvey distribute create --type token
resurvey distribute create --type panel --required-param pid --required-param wave
```

The URL points at `https://resurvey.teleios.au/d/<distributionId>`. Same URL stays stable across redeploys by default (in-flight respondent sessions stick to the release they started on). Use `resurvey deploy --fork` if the change is structurally breaking and old responses shouldn't be merged with new.

### Pull responses

```sh
resurvey responses summary
resurvey responses tail            # live event stream
resurvey responses pull --out responses.jsonl
```

## Schema reference

### `survey.config.ts`

```ts
export default {
  // What this survey is and what it's for. Read by Studio + the dashboard.
  name: "Customer NPS — Q1 2026",
  meta: {
    description: "Quarterly NPS with one open-ended follow-up. Audience: paying users; decision this informs: whether to invest the next quarter's roadmap in feature work or onboarding fixes.",
    estimatedDurationMinutes: 2,
  },

  // Defaults used by Studio when previewing locally.
  preview: {
    distributionType: "public",
    region: "NA",
    currency: "USD",
  },

  // Optional: notes for the author (cited sources, instrument provenance, etc.)
  // Not shown to respondents.
  notes: [
    "NPS wording: Reichheld 2003 (Net Promoter Score)",
    "Sample target: ~400 paying users; expect ~120 completes.",
  ],
};
```

### `src/survey.tsx`

A React default-exported component. Uses `useSurveyContext()` to read the session id, `submitSafe()` (or runtime block helpers) to write events. Branching, randomization, screener logic — all plain React state. Use runtime blocks for question UI.

Minimal shape:

```tsx
import { useState } from "react";
import { submitSafe, useSurveyContext, NPS, FreeText, Button, Card, Layout } from "@resurvey/survey-runtime";

export default function Survey() {
  const ctx = useSurveyContext();
  const [step, setStep] = useState<"intro" | "nps" | "why" | "done">("intro");
  // ...
}
```

The skill writes this file. It's allowed to add/remove steps, add question handlers, and edit JSX in the conventional places. It does not edit imports it didn't add, scaffolding, or runtime internals.

## Hard constraints

- **≤10 questions** by default — anything longer must be justified in `CRITIQUE.md`.
- **Every question passes the stress test** — read aloud, one breath, one thing being asked.
- **Use runtime blocks** for standard question types. Custom UI is the exception, not the default.
- **Never claim anonymity stronger than the distribution provides.** A `token` distribution is identified by token — say so.
- **Don't reimplement the event pipeline.** `trackAnswer`, `trackComplete`, and the runtime blocks handle batching, retry, dedup, beacon-on-unload, and revision tracking. Working around them loses data.
- **Don't touch `package.json`, `vite.config.ts`, `index.html`, or `src/main.tsx`.** Engine upgrades happen in the runtime repo and propagate to surveys on the next `resurvey deploy`.
