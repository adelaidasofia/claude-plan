---
name: plan
description: Temporal Agency skill that moves your to-dos into Google Calendar as time-boxed blocks, protects biological anchors (sleep, exercise, meals) first, separates maker from manager time, and tracks scheduled-vs-shipped as a credibility scorecard. Four modes: /plan week (Sunday planning), /plan morning (daily forward briefing + gratitude), /plan review (Friday scorecard + closure), /plan touch (reactive when a meeting request lands mid-day). Trigger when the user says /plan, "plan my week," "what's my day," "review the week," or when a new meeting request arrives and they ask whether to accept.
argument-hint: "[week | morning | review | touch], omit to be asked"
---

# /plan: Temporal Agency

A forcing function that moves planning from list-mode to calendar-mode. Reads from your to-do files, Google Calendar, and (optionally) journal/state signals. Writes calendar events with explicit human-in-the-loop approval, enforcing biological anchors first, maker blocks second, manager clusters third.

## Mode routing

Parse the first argument after `/plan`:

| Arg | Mode | When to run |
|---|---|---|
| `week` | Weekly planning ritual | Sunday afternoon, or any time you want to reset the week |
| `morning` | Daily forward briefing + gratitude | Start of your workday |
| `review` | Friday scorecard + closure | End of the workweek |
| `touch` | Reactive, a new meeting request landed | Anytime the user asks "should I take this meeting?" |
| _(no arg)_ | Ask which mode | Default |

If no arg, ask: "Which mode, week, morning, review, or touch?" Do not guess.

## First-run config

On first invocation, look for `[CWD]/.plan-prefs.md`. If it doesn't exist, walk the user through a one-time setup:

1. **Calendar account(s) to read:** Comma-separated list of Google Calendar accounts (e.g., `you@work.com, you@personal.com`). Read events from all; the union gives full availability.
2. **Calendar account to write:** Single account where new events get created. Defaults to the first account in (1).
3. **Timezone:** IANA name (e.g., `America/New_York`, `Europe/London`, `Asia/Tokyo`). Used for every `cal_create_event` call's `start`/`end` offset.
4. **Working hours:** Start and end of your default workday (e.g., `09:00` to `22:00`). Maker and manager blocks fit inside this window.
5. **Biological anchors:** Sleep window (e.g., `00:00`-`09:00`), exercise slot (e.g., `17:00`-`18:00` weekdays), meal windows (e.g., breakfast `09:30`-`10:30`, lunch `13:00`-`14:00`, dinner `18:30`-`19:30`). These are non-negotiables, booked first, every week.
6. **To-do source (optional):** Where do your tasks live? Common patterns: a single `TODO.md`, a folder of markdown files, or none ("I'll paste tasks each session"). If a path, the skill reads it on every `/plan week`.
7. **Bilingual support (optional):** Some calendars need event titles in different languages depending on attendee. List partner names + their preferred language (e.g., `Alex: es, Jordan: en`). When an event includes a listed partner, write the title + description in their language.

Save preferences to `[CWD]/.plan-prefs.md` as YAML frontmatter + a single markdown body. Don't ask again on future runs unless the user types `/plan reset`.

## Global rules (apply across all modes)

1. **Human-in-the-loop before every write.** Never call `cal_create_event`, `cal_update_event`, or `cal_delete_event` without showing the proposed change and getting explicit approval first. Hard rule.
2. **Timezone offsets on every event.** Every `start` and `end` field MUST end in an explicit offset (`-05:00`, `-04:00`, `Z`, etc.). Naive ISO strings get interpreted as UTC and land hours off. Use the configured timezone.
3. **No em dashes in calendar event titles.** They render badly across mobile and desktop calendar UIs. Use commas, colons, or periods.
4. **State-adaptive capacity (optional).** If the user has a self-reported energy or state signal (e.g., a journal frontmatter field like `energy: 1-5`, or a state tracker), scale bookable hours:
   - Energy 4-5 (high): 70% of working hours bookable
   - Energy 3 (mid): 65%
   - Energy 2 (low): 50%, prefer recovery + light tasks
   - Energy 1 (depleted): emergency mode, only sleep, exercise, one short maker block, meals
   Without an energy signal, default to 65%.
5. **Biological anchors first.** Sleep, exercise, meals get booked before any work block. If a work block conflicts with an anchor, the anchor wins by default, surface the conflict and ask before overriding.
6. **Maker vs manager separation.** Maker blocks (deep work) get contiguous 90-180 minute slots. Manager blocks (meetings, email, syncs) cluster into a single block per day. Don't interleave, context switches kill maker output.
7. **Implementation-intention event titles.** Format: `If [time] [day], then [action] at [location]`. Research-backed: implementation intentions roughly double follow-through vs. vague titles. Example: `If 11am Mon, then draft Q3 OKRs at desk` instead of `Draft OKRs`.
8. **Memory durability.** Anything you want to remember across sessions (scorecards, weekly plans, adherence data) MUST be written to a vault file. In-memory is not durable across new sessions.
9. **Zero hallucination on dates and events.** Never invent a calendar event from memory. Always call `cal_list_events` and quote what's actually on the calendar. If unsure about a date, ask.

## Default schedule template

A generic night-owl-friendly template. Adjust at first-run config to fit your chronotype.

| Time (local) | Block | Type | Notes |
|---|---|---|---|
| (sleep window) | Sleep | Non-negotiable | Hard cutoff at the end of the day. |
| Wake + 30min | `/plan morning` | Ritual | Gratitude + today's calendar + capacity math + top 3 priorities. |
| Wake + 30min to + 2hr | Slow start | Flex | Breakfast, coffee, light admin. Not maker time, nervous system still booting. |
| Mid-morning, 2hr block | **MAKER #1** | Deep work | Contiguous, no meetings. Auto-reject meeting invites. |
| 13:00-14:00 (or local lunch) | Lunch | Non-negotiable | Step away from work. |
| Afternoon, 2-3hr | MANAGER cluster | Shallow + social | Meetings, email, syncs. Book external meetings here. |
| Pre-evening | Exercise + recovery | Non-negotiable | 5x/week target. |
| Dinner window | Dinner | Non-negotiable | No work. |
| Evening, 2hr block | **MAKER #2** | Deep work | Optional second maker block, often creative work. |
| Pre-sleep | `/plan review` lite + journal | Ritual | 10-min reflection. |
| Wind-down | Wind down | Flex | Reading, social. No screens / scroll. |

The user can override any of this at `/plan week` approval. The template is a starting point, not the law.

## WEEK FLOW: `/plan week`

### Purpose
Produce a time-boxed weekly plan. Output: markdown proposal → user approval → calendar events written.

### Step-by-step

**1. Gather context (parallel reads):**
- Read the configured to-do source (file or folder). Pull P1/P2 items; note any with explicit due dates.
- `cal_list_events` for the upcoming Mon-Sun across all configured read accounts. Union results, dedupe by event ID.
- `cal_list_events` for the prior 7 days across all read accounts (completion data for the scorecard).
- If a state signal exists (journal frontmatter, mood tracker), read the latest entry.
- Read last week's scorecard at `[CWD]/Plans/YYYY-WW.md` if it exists.

**2. Capacity math:**
- Working hours = (working hours end - start) × 5 weekdays, minus meals, minus exercise, minus rituals.
- Apply capacity cap from Global Rule #4. Default 65% without a state signal.
- Subtract existing calendar commitments from capped capacity.
- Remainder = available for new blocks this week.

**3. Propose the week:**
Write a markdown block (do NOT write to calendar yet) with this structure:

```markdown
# Week of YYYY-MM-DD

**State:** [if a signal exists, summarize], capacity cap [%]
**Context:** [any anchor events the user mentioned: travel, deadline, deliverable]
**Exercise target:** [N] sessions. Last week: [X] sessions.

## Non-negotiables (booked first)
- Sleep: 7 nights × [N] hours
- Exercise: [N] sessions × [duration]
- Meals: 21 × [duration]
- Rituals: morning + evening blocks

## Maker blocks
- **MON 11:00-13:00**: If 11am Mon, then [project] [project tag]
- **MON 19:30-22:00**: If 7:30pm Mon, then [project] [project tag]
- [... ~10 across the week]

## Manager clusters
- **TUE 14:00-17:00**: Meetings cluster: [name] 2pm, [name] 3pm, slot 4pm
- **THU 14:00-17:00**: Team sync 2pm, email triage 3pm, slot 4pm
- [...]

## To-do blocks (from this week's P1s)
- **WED 11:00-13:00**: [task] (Source: [path]:[line])
  - Estimate: [hrs]. Padding (planning-fallacy coefficient × estimate): [adjusted hrs]

## Aged tasks (7+ days)
- "[task]", [N] days old. **Schedule [slot] or drop?**

## Capacity check
- Bookable: [N] hrs
- Proposed: [N] hrs
- Buffer: [N] hrs

## Conflicts flagged
- Tue 10am: existing event "[X]" conflicts with proposed [Y]
- Recommend: [resolution]
- Confirm before write? [y/n]

## Unscheduled important (this week's "can't fit")
- [tasks that didn't make it; prompt to push to next week or drop]
```

**4. Wait for approval.** The user may edit, swap, add, or reject. Do not write to calendar until they say "ship it" or equivalent.

**5. Write calendar events in batch.** For each approved block, call `cal_create_event` with:
- **Title:** implementation-intention format (Global Rule #7)
- **Description:** `Source: [vault file]:[line]\nType: [maker|manager|exercise|meal|ritual|buffer]\nEstimate: [N] hrs\nPlanning-fallacy coefficient: [last week's]\nState at plan: [signal]`
- **Calendar:** the configured write account
- **Reminders:** 10min before for meetings, 0min for rituals
- **Color (optional):** if your calendar supports it, use a consistent color per block type

**6. Write the week-plan markdown** to `[CWD]/Plans/YYYY-WW.md` for `/plan review` comparison later.

**7. Initialize the scorecard** at `[CWD]/Scorecard/YYYY-WW.md` with "scheduled" columns populated and "shipped" columns empty (Friday's `/plan review` fills them).

### Edge cases

- **No to-do source configured:** ask the user to paste their priorities for this week, then schedule from the paste.
- **Existing conflict with a non-negotiable:** surface the conflict, propose alternative slot, do not silently reschedule.
- **Aged task (14+ days):** hard schedule-or-drop prompt. No third option. Stale tasks erode trust in the system.
- **Travel week:** prompt the user for trip timezone at start; adapt template; biological anchors stay anchored to the trip TZ.

## MORNING FLOW: `/plan morning`

### Purpose
Daily forward briefing + gratitude. Check today's plan, adapt to overnight changes, surface missed blocks, kill 2-minute tasks inline.

### Step-by-step

**1. Check the week plan exists.** Read `[CWD]/Plans/YYYY-WW.md`. If missing, hard block:
> "No plan for this week. Run `/plan week` first (~15 min) or tell me to cascade a minimum-viable plan (sleep + exercise + 1 maker block/day)."

**2. Context:**
- `cal_list_events` for today 00:00-23:59 across all read accounts, dedupe by event ID
- Read yesterday's state signal if any
- Read this week's plan
- Check the rolling scorecard for adherence numbers

**3. Greet + gratitude:**
> "Good morning. Today is [day, date]. State signal: [X if exists]. Sleep: [Y] hours.
> 
> **Three-line gratitude:** What are 2-3 things you're grateful for this morning?"

Wait for input. Save the lines to the daily log under `## Gratitude`.

**4. Show today's blocks:**
Render today in order, marking [PLANNED] vs [CONFIRMED] (already on calendar):

```
## Today's plan (YYYY-MM-DD)
[time]  Slow morning
[time]  MAKER: [task] [PLANNED]
[time]  Lunch
[time]  Call: [name] [CONFIRMED]
[time]  Manager cluster [PLANNED]
[time]  Exercise [PLANNED]
[time]  Dinner
[time]  MAKER: [task] [PLANNED]
[time]  Journal
[time]  Wind down
```

**5. Capacity check:**
- Working hours today: [N]
- Booked: [N]
- Buffer: [N]

**6. Top 3 rocks today:**
> "Of today's blocks, which 3 *must* happen for today to feel like a win?"

Save the rocks to the scorecard for this week.

**7. 2-minute triage:**
Scan the to-do source for items tagged `[2min]` or similar. If any, propose handling them in the next 10 minutes inline (no calendar event needed).

**8. Weekly anchor reminder.** If today is Mon/Wed/Fri, remind of the next anchor event (deadline, presentation, travel) within 14 days.

## REVIEW FLOW: `/plan review`

### Purpose
Friday scorecard + closure. Close open loops mentally before the weekend (the Zeigarnik effect, unfinished tasks intrude on rest).

### Step-by-step

**1. Read the week plan** and `cal_list_events` for the past Mon-Fri across read accounts.

**2. For each block in the week plan, mark shipped or missed:**
For maker blocks, ask: "Did the block happen as planned, partial, or skipped?" Record. Compute:
- Maker adherence: `shipped / planned`
- Manager adherence: same
- Anchor adherence (sleep, exercise, meals): same
- Overall: weighted average

**3. Update the scorecard** at `[CWD]/Scorecard/YYYY-WW.md`:

```markdown
# Scorecard YYYY-WW (Mon DD - Sun DD)

| Category | Planned | Shipped | Adherence |
|---|---|---|---|
| Maker blocks | 10 | 7 | 70% |
| Manager clusters | 5 | 5 | 100% |
| Exercise | 5 | 4 | 80% |
| Sleep (avg hours) | 8 | 7.2 | 90% |
| Meals (skipped) | 0 | 1 | 95% |
| **Overall** |, |, | **84%** |

## Wins
- [shipped block 1]
- [shipped block 2]

## Misses + reason
- [missed block 1]: [reason, meeting ran over, energy crash, etc.]

## Drift detected
- [pattern, e.g., "Maker #2 missed 3x; evening blocks need shorter duration"]

## Adjustments for next week
- [change to template, capacity cap, or ritual]
```

**4. Zeigarnik closure.** Surface any P1 items still open from this week. For each:
> "[task] is still open. Push to next week, drop, or close in the next 30 min?"

**5. Friday wind-down ritual:**
> "Three things you're proud of from this week:
> 1. ?
> 2. ?
> 3. ?
> 
> Anything you want to hand off to your next-week self?"

Save to a "Friday close" log at `[CWD]/Logs/YYYY-MM-DD-friday-close.md`.

## TOUCH FLOW: `/plan touch`

### Purpose
A new meeting request landed. Decide whether to accept, propose an alternative, or decline.

### Step-by-step

**1. Get the meeting details** from the user:
- Who's asking
- Proposed times
- Duration
- Topic (1 sentence)
- How important the relationship is (1-5)

**2. Check current week plan and calendar:**
- Read this week's `Plans/YYYY-WW.md`
- `cal_list_events` for the proposed window across read accounts

**3. Three-question gate:**
- **Will this meeting move a P1 forward?** If no and the relationship is <3, lean decline.
- **Does it fit a manager cluster?** If yes, propose that slot. If it would land in maker time, propose moving to manager time.
- **Is the asker someone whose time is more constrained than yours?** If yes (e.g., a senior advisor offering 30 minutes), bias toward accepting their slot.

**4. Propose a response:**
Three flavors:
- **Accept (preferred slot):** Draft a one-line confirmation with the implementation-intention title.
- **Counter (manager-cluster slot):** Draft a polite counter-offer pointing at your manager-cluster window.
- **Decline (politely):** Draft a no-thanks that opens a door for asynchronous follow-up.

Show all three. Wait for user choice.

**5. On accept:** create the calendar event using the standard write flow. Add it to this week's plan markdown so `/plan review` Friday sees it.

## Why this skill exists

Most planning tools live as lists. Lists don't care about time. A 14-item to-do list is the same length whether you have 8 hours or 8 days, which is why most lists never get done, they don't fit.

This skill enforces fit. Every task either fits in a real calendar block or it doesn't get scheduled. Tasks that don't fit get triaged, dropped, or pushed. The scorecard surfaces the gap between scheduled-self and shipped-self, which is the only signal that tells you whether your planning is calibrated to your actual capacity.

The biological-anchors-first rule comes from the idea that sleep, exercise, and meals are non-negotiable inputs to cognitive output. Booking them first means work blocks happen with rested brain, not at the cost of rested brain. Maker/manager separation comes from Paul Graham's essay of the same name: deep work needs contiguous time; meetings shred it.

Human-in-the-loop before every calendar write is paranoia about wrecked plans. Calendars are public commitments. A skill that silently writes events can blow up your week before you've reviewed it.
