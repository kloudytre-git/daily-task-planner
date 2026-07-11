# 🗓️ AI Daily Task Planner — Built with AWS PartyRock

> An interactive, no-code generative-AI app that turns a raw to-do list into a prioritized, time-blocked daily schedule.

**🔗 Live App:** [`<https://partyrock.aws/u/kloudy/kMa2P8QFI/Daily-Task-Scheduler>`](#)
**📸 Example Run (Snapshot):** [`<https://partyrock.aws/u/kloudy/kMa2P8QFI/Daily-Task-Scheduler/snapshot/l0Sd-Vn0J>`](#)

<!-- 🎬 HERO DEMO — attach a 10–15 second screen recording of the full flow: paste tasks → run → schedule appears.
     Drop your file at docs/demo.gif and it will render automatically. -->
![App demo](docs/schedule-demo.gif)

---

## 📌 Table of Contents

- [Overview](#overview)
- [What It Does](#what-it-does)
- [Tech Stack](#tech-stack)
- [Architecture / How It Operates](#architecture--how-it-operates)
- [Prompt Engineering](#prompt-engineering)
- [Results](#results)
- [How to Use It Yourself](#how-to-use-it-yourself)
- [How to Rebuild It](#how-to-rebuild-it)
- [Limitations](#limitations)
- [What I Learned](#what-i-learned)
- [Future Work](#future-work)

---

## Overview

This project is an AI-powered **Daily Task Planner** built entirely on **AWS PartyRock**, a no-code generative-AI playground powered by Amazon Bedrock. The user pastes in an unstructured to-do list along with how many hours they have and their current energy level; the app returns a realistic, prioritized, time-blocked schedule, a tailored motivational tip, and a chat assistant for reshuffling the plan on the fly.

The goal was to explore how far **prompt engineering alone** — with zero code, zero servers, and zero infrastructure — can go toward building a genuinely useful productivity tool.

---

## What It Does

| Input | → | Output |
|---|---|---|
| 📝 Today's task list (free-form) | → | ⏱️ Time-blocked schedule with start–end times |
| 🕐 Hours available | → | 📊 Tasks front-loaded by impact & energy |
| ⚡ Energy level (low / med / high) | → | ☕ Built-in breaks every ~90 minutes |
| | → | 📋 "Deferred to tomorrow" list if tasks overflow |
| | → | 💬 Chat assistant to adjust the plan |
| | → | ✨ One tailored motivational tip |

---

## Tech Stack

| Component | Purpose |
|---|---|
| **AWS PartyRock** | No-code app builder / hosting |
| **Amazon Bedrock** | Foundation models powering generation |
| **Whiskers (PartyRock AI builder)** | Generated the initial app scaffold from a plain-English spec |
| **PartyRock Widgets** | User Input, AI Text Generation, Chatbot |
| **Prompt Engineering** | The actual "logic" of the app |

> No code. No AWS account required. No servers, databases, or deployment pipeline.

---

## Architecture / How It Operates

A PartyRock app is a **graph of connected widgets**. Input widgets collect data; AI output widgets reference those inputs via `@mentions` and send a prompt to a Bedrock foundation model, which returns generated text.

```
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│  📝 Tasks       │  │  🕐 Hours       │  │  ⚡ Energy      │
│  (User Input)   │  │  (User Input)   │  │  (User Input)   │
└────────┬────────┘  └────────┬────────┘  └────────┬────────┘
         │                    │                    │
         └────────────┬───────┴────────────────────┘
                      │  @Tasks  @Hours  @Energy
                      ▼
         ┌────────────────────────────┐
         │  🤖 Amazon Bedrock Model   │
         │  + engineered prompt       │
         └────────────┬───────────────┘
                      │
        ┌─────────────┼──────────────┐
        ▼             ▼              ▼
┌──────────────┐ ┌──────────┐ ┌──────────────┐
│ ⏱️ Schedule  │ │ ✨ Tip   │ │ 💬 Chatbot   │
│  (AI Output) │ │(AI Output)│ │  (adjusts)  │
└──────────────┘ └──────────┘ └──────────────┘
```

**Execution flow:**
1. User fills the three input widgets.
2. User hits **▶ Run** on the schedule widget.
3. PartyRock substitutes the `@referenced` input values into the widget's prompt.
4. The prompt is sent to the selected Bedrock foundation model.
5. Generated text renders back into the output widget.
6. The chatbot widget can then take follow-up requests against that plan.

> ⚠️ **Important:** This is an **on-demand interactive app**, not a background job. PartyRock has no scheduler or server, so it does not auto-run each morning — the user opens it and runs it. See [Future Work](#future-work) for the true-automation path.

---

## Prompt Engineering

The prompts *are* the application logic. Below are the instructions driving each AI widget.

### 🧠 Schedule Generator (core widget)
```
You are a productivity coach. Using the tasks in @Tasks, the time in @Hours, and
the energy level in @Energy, create a realistic time-blocked schedule for today.

Rules:
- Front-load the highest-impact or hardest tasks, especially if energy is high.
- Group similar tasks together.
- Insert a 5–10 minute break roughly every 90 minutes.
- Never schedule more work than the available hours allow. If tasks don't fit,
  list what to defer to tomorrow.

Format as a clean timetable with start–end times, followed by a short
"Deferred to tomorrow" section if needed.
```

**Design decisions:**
- **Energy-aware ordering** — hard tasks go early when energy is high, mirroring how cognitive load actually decays across a day.
- **Forced deferral** — without an explicit overflow rule, the model happily crams 9 hours of work into a 6-hour day. Making deferral mandatory is what made the output trustworthy.
- **Explicit formatting rules** — keeps output scannable instead of a wall of prose.

### ✨ Motivational Tip
```
Write one short, specific motivational tip (max 2 sentences) tailored to a day
containing these tasks: @Tasks. Keep it warm and practical, not generic.
```

**Design decisions:**
- Hard 2-sentence cap prevents generic filler.
- Referencing `@Tasks` forces specificity instead of fortune-cookie platitudes.

### 💬 Chat Assistant
```
You help the user adjust the schedule produced above. When they ask to reshuffle,
shorten, or swap tasks, return an updated timetable. Be concise.
```

**Design decisions:**
- Scoped narrowly to schedule edits so it doesn't drift into open-ended chat.

### Model Selection

| Widget | Model Used | Why |
|---|---|---|
| Schedule | `<MODEL>` | Reasoning quality matters most; worth the higher credit cost |
| Tip | `<MODEL>` | Lightweight task — a cheaper model is sufficient |
| Chatbot | `<MODEL>` | Mid-tier; needs to follow the existing plan, not invent one |

---

## Results

### Before → After Prompt Refinement

The same input, run against the first-draft prompt and the final prompt:

| ❌ v1 prompt | ✅ Final prompt |
|---|---|
| Ignored the hours limit — scheduled ~9 hours of work into a 6-hour day | Respects the cap and explicitly defers overflow |
| Returned a flat, unordered list | Returns real time blocks with start–end times |
| No breaks | Inserts breaks every ~90 minutes |
| Treated all tasks as equal priority | Front-loads high-impact work based on energy level |

### Sample Output

```
<PASTE A REAL GENERATED SCHEDULE HERE>
```

**📸 Live snapshot:** [`<PASTE SNAPSHOT LINK>`](#)

---

## How to Use It Yourself

1. Open the [live app link](#).
2. Paste your task list into **Today's Tasks** (one per line).
3. Enter **Hours Available** and pick your **Energy Level**.
4. Hit **▶ Run** on the schedule widget.
5. Use the **chat assistant** to reshuffle anything you don't like.

> 💡 You don't need my permission or credits — PartyRock gives every user a free daily credit allowance, and you spend your own. You can also hit **Remix** to fork it into your own version.

---

## How to Rebuild It

Full step-by-step build instructions are in **[`PartyRock-Daily-Task-Planner-Guide.md`](./PartyRock-Daily-Task-Planner-Guide.md)**.

Quick version:
1. Sign in at `partyrock.aws` (Google / Apple / Amazon login).
2. Click **Create an app** → describe the planner to **Whiskers**.
3. Edit each widget's prompt (see [Prompt Engineering](#prompt-engineering)).
4. Test, iterate, and **Snapshot** a good run.
5. **Share** → set visibility → copy link.

---

## Limitations

Being honest about constraints is part of the engineering.

| Limitation | Impact |
|---|---|
| ⏰ **No scheduler / automation** | PartyRock has no cron or server — the app can't auto-run each morning. It's on-demand. |
| 🌐 **English only** | PartyRock is designed for English-language prompts. |
| ⬇️ **No file downloads** | Schedules can't be exported to a file; copy/paste or snapshot instead. |
| 🎟️ **Daily credit limit** | Free usage resets daily; heavy iteration can exhaust it. |
| 💾 **No persistent state** | The app doesn't remember yesterday's tasks or track completion. |
| 🎲 **Non-deterministic output** | The same input can produce slightly different schedules run to run. |

---

## What I Learned

- **Prompt specificity is everything.** Vague instructions produced vague, unusable schedules; adding hard constraints (hours cap, forced deferral, break cadence) is what made the output trustworthy.
- **Constraints must be explicit.** The model will happily overschedule a day unless you *tell it* that overflow must be cut.
- **The AI builder isn't infallible.** Whiskers occasionally described features that don't exist (like file downloads), which taught me to verify rather than trust.
- **Model choice is a cost/quality tradeoff.** More capable models write better schedules but burn credits faster.
- **Validate the prompt before building the infrastructure.** Prototyping in PartyRock proved the logic worked before committing to a real AWS pipeline.

---

## Future Work

### Extending the PartyRock app
- [ ] Add a **Document** widget so users can upload a calendar export and have the plan work around fixed meetings.
- [ ] Add an **end-of-day reflection** output widget.
- [ ] Build a **weekly** planner variant.

### Phase 2 — True automated daily scheduling

PartyRock can't self-trigger. To get a schedule **emailed to me automatically at 7:00 a.m. every day**, the same prompt would move into a real AWS pipeline:

```
Amazon EventBridge Scheduler  →  AWS Lambda  →  Amazon Bedrock  →  Amazon SES
       (daily 7am cron)          (function)    (same prompt!)      (email me)
                                      ↕
                                Amazon DynamoDB
                             (stores recurring tasks)
```

The PartyRock build serves as the **prototype that validates the prompt** before committing to that infrastructure.

---

**Built by:** `<YOUR NAME>` · **Date:** `<DATE>` · **Platform:** AWS PartyRock (Amazon Bedrock)
