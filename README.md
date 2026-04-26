# claude-plan

A Claude Code plugin that turns your to-do list into a real, time-boxed week. Reads your tasks and Google Calendar, proposes a plan that protects sleep / exercise / meals first, separates maker from manager time, and tracks scheduled-vs-shipped as a credibility scorecard.

---

## What it does

Four modes you invoke as a slash command:

- **`/plan week`** — Sunday (or any time you want to reset). Reads your to-dos and calendar, proposes a markdown week plan, waits for your approval, then writes calendar events with implementation-intention titles like *"If 11am Mon, then draft Q3 OKRs at desk"*.
- **`/plan morning`** — Start of your day. Pulls today's events across configured calendar accounts, asks for a 3-line gratitude, surfaces capacity math + top 3 priorities, kills 2-minute tasks inline.
- **`/plan review`** — Friday. Computes adherence (planned vs shipped), updates the scorecard, surfaces drift patterns, runs a 3-things-I'm-proud-of close.
- **`/plan touch`** — Reactive. A new meeting request landed; decide accept / counter / decline using a three-question gate.

The skill never writes to your calendar without explicit approval. Every event includes an explicit timezone offset (no naive ISO strings — those silently land hours off).

---

## Why use it

Most planning tools live as lists. Lists don't care about time. A 14-item to-do list is the same length whether you have 8 hours or 8 days, which is why most lists never get done — they don't fit.

This skill enforces fit. Tasks that fit in a real block get scheduled. Tasks that don't fit get triaged, dropped, or pushed. The scorecard surfaces the gap between scheduled-self and shipped-self, which is the only signal that tells you whether your planning is calibrated to your actual capacity.

---

## Install

Inside Claude Code, run:

```
/plugin install claude-plan@adelaidasofia/claude-plan
```

Or clone manually:

```bash
git clone https://github.com/adelaidasofia/claude-plan.git ~/.claude/plugins/claude-plan
```

---

## Setup

The skill needs the [google-workspace MCP](https://github.com/taylorwilsdon/google_workspace_mcp) (or any Google Calendar MCP that exposes `cal_list_events`, `cal_create_event`, `cal_update_event`, `cal_delete_event`) connected to your account before it can do anything.

On first invocation, the skill walks you through a one-time setup:

1. Calendar account(s) to read (comma-separated)
2. Calendar account to write (single)
3. Timezone (IANA, e.g. `America/New_York`)
4. Working hours (start + end of your default workday)
5. Biological anchors (sleep window, exercise slot, meal windows)
6. To-do source (optional — file path, folder, or "I'll paste each session")
7. Bilingual support (optional — partner names + their preferred language)

Preferences save to `[CWD]/.plan-prefs.md`. Type `/plan reset` to redo setup.

---

## Built-in rules

- **Human-in-the-loop before every write.** No calendar event ever lands without your approval.
- **Timezone offsets on every event.** Naive ISO strings get interpreted as UTC; the skill always passes an explicit `-05:00` / `Z` / etc.
- **Biological anchors first.** Sleep, exercise, meals get booked before any work block.
- **Maker vs manager separation.** Deep work blocks get contiguous 90-180 minute slots. Meetings cluster into a single block per day.
- **Implementation-intention titles.** Format `If [time] [day], then [action] at [location]`. Research-backed: ~2x follow-through vs. vague titles.
- **Zero hallucination on dates.** Never invent a calendar event from memory. Always call `cal_list_events` and quote actual events.

---

## Files the skill writes

- `[CWD]/.plan-prefs.md` — your config (one-time)
- `[CWD]/Plans/YYYY-WW.md` — the week plan (one per ISO week)
- `[CWD]/Scorecard/YYYY-WW.md` — scheduled vs shipped (one per ISO week)
- `[CWD]/Logs/YYYY-MM-DD-friday-close.md` — Friday review entries

You control whether any of these get committed to your vault or repo.

---

## Why this skill exists

The skill enforces what every productivity book repeats but most tools don't operationalize: *fit before priority*. You can't do everything important; you can only do what fits. The scorecard makes that visible. The state-adaptive capacity cap (optional, hooks into a journal frontmatter field if you have one) keeps you from booking 70% of working hours when you slept 4 hours and your nervous system is hammered.

The maker-vs-manager rule comes from Paul Graham's essay of the same name. The implementation-intention research comes from Peter Gollwitzer's work — adding "if [time], then [action]" to a goal roughly doubles follow-through. The biological-anchors-first rule comes from the realization that sleep, exercise, and meals are inputs to cognitive output, not luxuries booked after work fits.

---

## License

MIT. See [LICENSE](LICENSE).
