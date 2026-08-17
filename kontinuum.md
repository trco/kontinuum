# Kontinuum — 24/7 development, wired around GitHub

> **Status:** initial specification (draft v0, 2026-08-16). Describes the *concept* and its shared
> contract, not code. Kinetik ([`kinetik-mvp.md`](https://github.com/trco/kinetik/blob/master/docs/kinetik-mvp.md)) is the reference **consumer**;
> this doc is the layer *around* it — the producers, and the Convention that lets everything
> interoperate through GitHub without knowing about each other.

---

## 1. One-sentence framing

**Kontinuum is a way to run software development continuously: many independent producers (things that
create tasks) and many independent consumers (things that do them — developers and Kinetik), wired
around GitHub as the center. Its heart is the Convention — a small shared contract every tool speaks,
so no tool ever depends on another.**

It is a *concept you wire*, not a *tool you run*. Nothing has a central brain; coordination happens
because everyone speaks the same GitHub state.

---

## 2. Why it's shaped this way

Two beliefs drive every decision below.

- **Human attention is the scarce resource, not AI labor.** AI in the middle is cheap and near-
  infinite. The bottleneck is human judgement at two points: **triage-in** (what's worth doing, and by
  whom) and **merge-out** (only a developer ships). The whole system exists to spend those two moments
  well, and let AI be abundant in between.
- **GitHub is already the platform-agnostic hub.** Issues, Projects, Repos, and PRs are a shared state
  store with a permission model and an audit trail. We don't build a hub — we agree on how to use one.

The style is **choreography, not orchestration** (think Unix pipes: small independent tools composed
through a shared medium). The shared medium is GitHub. Slack, meeting notes, or an AI planner are just
*sources* — one possible input, never a hard dependency.

---

## 3. The parts

```
   PRODUCERS                         GITHUB  (the center)                    CONSUMERS
   (create tasks)                    (shared state + truth)                  (do tasks)

   PM plugin  ─┐                ┌──────────────────────────────┐            ┌─ developer
   researcher ─┼──  write  ──►  │  Issues · Projects · Repos · PRs │  ──► read ┤
   maintainer ─┤   real issue   │  labels = the interface          │            └─ Kinetik
   AI planner ─┘                └──────────────────────────────┘
                                        ▲
                                   THE CONVENTION
                        (registry · task shape · label lifecycle)
                        the only thing every part agrees on
```

- **Substrate (the center): GitHub.** Not built — *used*. Shared state and the meeting point. Every
  other part talks only to GitHub, never to each other.
- **The Convention.** The single shared contract (Section 4). It is the concept. Everything conforms to
  it; it conforms to nothing. It can start as this one page, not code.
- **Producers.** Independent, mostly hand-triggered tools that *write* tasks: a PM plugin (Slack thread
  → task), the existing researcher and maintainer plugins, an AI spec/plan preparer. None knows about
  another producer or about any consumer.
- **Consumers.** Independent tools/people that *read* tasks and move them toward a merged PR:
  **developers** (manual) and **Kinetik** (automated). They don't know or care which producer made a
  task.

**The property that makes it a concept, not a tool:** every part depends only on the Convention and
GitHub — never on another part. Add a producer, drop a consumer, swap Slack for Asana: nothing else
changes. The only real prerequisite is the Convention, and it's the cheapest part.

---

## 4. The Convention (the contract)

Three thin agreements. All GitHub-native — rules, not machinery.

### 4.1 Registry — *where does a task go?*

A small **registry file** in a known repo maps sources to destinations:

```
slack_channel_id  →  github_project  →  default_repo(s)
```

Keyed by **IDs, not names**, so renames never break the wiring. A producer reads it to know where to
drop a task. It is the single source of truth for the wiring.

### 4.2 Task shape — *what is this, and who acts on it?*

A task is a **real GitHub issue** (never a Project *draft* — drafts have no repo and no labels, so
Kinetik can't see them). It carries:

- **type** — `bug` / `improvement` / `support` / `spec-task` (label)
- **source** — which producer/channel made it (label), for traceability
- **acceptance criteria** — a required section in the issue body. *This is the field that makes or
  breaks automated execution — see Open Questions.*
- **executor** — decided at triage by a human: Kinetik, or a developer.
- **status** — the label lifecycle below.

### 4.3 Label lifecycle — *how it moves, and who may move it*

The executor choice reduces to **one fork: does the issue get `kinetik:ready` or not?** That single
label is the switch between the automated lane and the manual lane.

Kinetik's labels already exist and are defined in code
([`kinetik/github.py`](https://github.com/trco/kinetik/blob/master/kinetik/github.py), [`kinetik-mvp.md` §schema](https://github.com/trco/kinetik/blob/master/docs/kinetik-mvp.md)):

- **State (mutually exclusive** — setting one strips the others**):**
  `kinetik:needs-triage` → `kinetik:ready` → `kinetik:claimed` → `kinetik:pr-open`, plus
  `kinetik:blocked` for escalation.
- **Human overrides (separate):** `kinetik:hold` (stop this issue), `kinetik:paused` (kill switch
  for the whole repo), `kinetik:plan-first` (open a plan-only PR and wait for a human to approve).

**Two sources of truth, kept apart:**

- **Labels** = the human ↔ system interface. Queue it, stop it, approve a plan. Set them straight from
  the Project board.
- **Claim log** (append-only issue comments, [ADR 0001](https://github.com/trco/kinetik/blob/master/docs/living-docs/adr/0001-claim-log-in-github-issue-comments.md))
  = machine ↔ machine ownership, so two Kinetik runners never collide. `kinetik:claimed` is only a
  human-readable *projection* of it. Producers and triage **never touch the claim log** — it's below
  the Convention.

**The one hard rule:** an automated consumer may move an issue *up to* `pr-open`, never past it. **Only
a developer merges.** That single constraint is the whole safety model.

---

## 5. Two stories

Same bug, both lanes, ending at a PR a human merges.

### Story A — a bug fixed by Kinetik

> A customer reports a crash in the `#support` Slack channel. A developer triggers the PM plugin on the
> thread. The plugin reads the **registry** (`#support → the Payments project → payments repo`), writes
> a **real issue** with a clear *acceptance criteria* section ("crash no longer occurs on empty cart;
> add a regression test"), and — confident it's actionable — labels it `kinetik:ready`.
>
> Kinetik, polling that repo, sees `kinetik:ready`, checks no `hold`/`paused` is set, and **claims**
> the issue (writes a claim-log entry; the label projects to `kinetik:claimed`). It works in a
> sandbox, opens a PR, and flips the label to `kinetik:pr-open`. A developer reviews and merges. The
> issue closes and drops out of the queue. **Human effort: two moments — triage and merge.**

### Story B — the same bug, done by a developer

> This one is subtle, so triage marks **executor = developer** — meaning it simply *doesn't* get
> `kinetik:ready`. Because that label is absent, **Kinetik never sees it.**
>
> A developer assigns themselves, moves the Project **Status** column by hand (New → In progress),
> writes the fix, and opens the PR themselves (Status → Review). Another developer merges.

**What's shared:** same issue, both end at a PR, merge is human-only in both. **What differs:** Lane A's
lifecycle lives in `kinetik:*` labels (machine-driven); Lane B's lives in the Project Status column
(human-driven). The fork was one label.

---

## 6. Deliberately out of scope (for now)

- **Custom overview UI.** GitHub Projects views + Insights already show cross-project/team state. Build
  UI only if GitHub genuinely can't.
- **A generic multi-source abstraction layer.** Agnosticism falls out of the task-shape contract — it's
  "more producers," not an upfront framework. Build GitHub-native with Slack as the first source.
- **Observer / auto-trigger producers, dedup, rate limits.** Producers are hand-triggered for now.
  Defer until one actually watches a channel unattended.
- **Review-throughput tooling.** The human merge bottleneck is real but it's a team-process problem, not
  a wiring problem. Named, not designed.

---

## 7. Open questions (to decide before the first producer ships)

1. **Producer output quality — the make-or-break.** What is the *minimum* issue template that lets a
   consumer (especially Kinetik) close an issue review-ready? The acceptance-criteria section is the
   highest-leverage thing left to pin down. Garbage in still breaks the 24/7 promise.
2. **Registry contents.** The actual `slack channel ↔ project ↔ repo` mappings. Small, but the first
   producer can't drop a task without it.
3. **Executor suggestion.** Confirmed: a human picks the executor at triage. Later question — may a
   producer *pre-suggest* it (leaving the human to confirm), or is the choice always human-first?
