# Agentic OS: agent orchestration platform

My field is financial analysis, business analysis, and operations: two USF business degrees (B.S. Personal Financial Planning and B.S. Marketing), relationship banking at Truist, and service operations at Geek Squad, with no formal software training. Agentic OS is the clearest evidence of how I handle process and data: a self-hosted dashboard that schedules, launches, and monitors my own automated jobs, records an outcome for every run, and routes future work by measured success rate rather than fixed rules. The jobs are AI agent runs driven through Claude Code, and I design, build, and test the system myself, with the same AI coding assistance for implementation. It is about 13,800 lines of TypeScript behind 31 API routes and 17 SQLite tables, running on a server I administer myself. The application source stays private because the running instance is wired into my personal data (calendar, health logs, a markdown knowledge base). What transfers is the process, so this page is the process: how the system works, how I build, what broke, and what I changed because of it.

Portfolio: [bryanwudarsky-lab.github.io](https://bryanwudarsky-lab.github.io) | [linkedin.com/in/bryanwudarsky](https://www.linkedin.com/in/bryanwudarsky)

| Measure | August 2026 snapshot |
|---|---|
| TypeScript | about 13,800 lines |
| API route handlers | 31 |
| Page routes | 20 |
| SQLite application tables | 17 |
| Recorded runs | 20, at a 60 percent success rate |
| Projects tracked | 22 |

## What the system does

Every run is a child process. A skill is a markdown file with frontmatter (routing keywords, a max duration) and a prompt body with {task} and {context} placeholders. Starting a run launches a command-line tool as a subprocess emitting a JSON event stream, parses that stream line by line, and pushes events to the browser over Server-Sent Events, so I can watch any run live. Concurrency is hard-capped so a burst of scheduled work cannot claim the whole machine.

A workflow chains skills into a dependency graph stored as JSON. Each step names the steps it waits on, everything unblocked fires in parallel, and an upstream step's result is interpolated into the downstream step's prompt, capped at 2,000 characters so a chatty step cannot crowd out the instructions. Definitions are validated at save time, including a graph cycle check, because rejecting a cycle when it is written beats discovering it mid-run. Retry rebuilds only the failed step and whatever was skipped downstream of it, so recovering from a mid-graph failure costs only the work below the break. A scheduler ticks once a minute against cron expressions and can fire a single skill or an entire workflow.

One design decision carries most of the economics: runs launch as subprocesses of a command-line tool rather than as direct calls to a metered API. Every run gets a full tool surface on day one (file reads and writes, shell, search) at zero incremental cost, because runs draw on a subscription I already pay for instead of per-use pricing. A system built for unattended scheduled work only earns its keep if running it daily costs nothing extra.

The infrastructure is mine end to end. The app deploys with Docker to a TrueNAS server, the knowledge base syncs automatically between my laptop and the server, remote access goes over Tailscale (WireGuard) with nothing exposed to the public internet, and the dashboard installs as a PWA on my phone.

## How I build

The code matters less than the workflow that produced it, because the workflow is what I would bring to a team, whether the deliverable is a model, a report, or a system. Every project I run follows the same loop, and this platform exists to automate the parts of that loop that used to live in my head.

I write the plan first. The platform came out of a written master plan with seven phases, each phase shipping working features end to end and getting verified in the running app before the next began. I own the spec, the reviews, and the verification.

Within a phase, the structure is adversarial on purpose:

1. A pre-code review panel, each reviewer briefed to attack the spec from a different angle, runs before any code exists.
2. Build tracks run in parallel, each with a capped scope and an ownership map naming the files it may touch.
3. An integrator merges the tracks and runs the automated test gate.
4. A post-code review panel hunts for the defects the tests cannot see.
5. I verify the gate and the repository state myself before anything counts as done.

The panels earn their place with findings. In one recent phase, the pre-code panel returned 19 amendments and 2 blocking-class defects against a spec I had considered ready, and after the build, the post-code review caught 4 more blocking defects that 17 passing spec files had missed. A passing suite proves the assertions I thought to write; the adversarial pass exists for everything I did not think to assert.

The platform also measures the loop instead of trusting it. Its ledger currently holds 20 recorded runs at a 60 percent success rate. That number stays on the dashboard, next to per-skill rates with run counts beside them, because a rate over 3 runs and a rate over 17 runs are different kinds of number, and because an automation system nobody measures quietly becomes a liability.

## The evaluation loop

Every run ends as one immutable row in an outcome ledger. Nothing updates a row and nothing deletes one, so the same query over the same window always returns the same answer. Five paths write to it, four of them automatic:

- exit code 0 writes success
- a non-zero exit writes failure
- a spawn error also writes failure, because from my side the run produced nothing either way
- cancellation writes cancelled
- a thumbs button writes thumbs_up or thumbs_down, with an optional comment

Because the automatic paths fire from the process lifecycle, coverage does not depend on my remembering to rate anything. A run that finishes at 3 a.m. on a schedule lands in the ledger with the same fidelity as one I launched by hand. Success rate deliberately counts only success, failure, and cancelled in its denominator: a thumbs row rates a run rather than adds one, and counting it would let a single well-received run look like three.

A scoring function reduces each skill's history to one number. An explicit human rating moves the score twice as far as an exit code: an exit code proves the process terminated, while a thumbs press means a person judged the output, and that signal is scarcer.

```ts
export function getOutcomeScores(): Map<string, number> {
  const stats = getSkillStats();
  const map = new Map<string, number>();
  for (const s of stats) {
    const sample = s.total_runs + s.thumbs_up + s.thumbs_down;
    if (sample === 0) {
      map.set(s.agent_type, 0);
      continue;
    }
    const successWeight = s.successes - s.failures;
    const feedbackWeight = (s.thumbs_up - s.thumbs_down) * 2;
    map.set(s.agent_type, (successWeight + feedbackWeight) / sample);
  }
  return map;
}
```

Those scores reach the router at exactly one point: the tie-break. A task submitted without an explicit skill gets keyword-scored against every installed skill on word boundaries. A clear top scorer wins outright, and no match at all falls back to the first skill in the list, a deterministic default rather than a guess. A tie at the top hands the decision to the ledger, and if the ledger query fails, routing degrades to the first tied candidate instead of failing the submission. This is the resolution path after the keyword scores are computed:

```ts
  const maxKeyword = Math.max(...scores.map(s => s.keywordScore));
  if (maxKeyword === 0) return skills[0];

  const tied = scores.filter(s => s.keywordScore === maxKeyword);
  if (tied.length === 1) return tied[0].skill;

  let outcomeScores: Map<string, number>;
  try {
    outcomeScores = getOutcomeScores();
  } catch {
    return tied[0].skill;
  }

  let best = tied[0];
  let bestOutcome = outcomeScores.get(best.skill.id) ?? 0;
  for (const candidate of tied.slice(1)) {
    const candidateOutcome = outcomeScores.get(candidate.skill.id) ?? 0;
    if (candidateOutcome > bestOutcome) {
      best = candidate;
      bestOutcome = candidateOutcome;
    }
  }
  return best.skill;
```

Placing history at the tie-break and nowhere else was deliberate. Keyword evidence is a direct signal about this specific task; outcome history is an indirect signal about the skill in general. Letting the indirect signal outrank the direct one would drift routing toward whichever skill performed well recently, regardless of whether it fit the request.

## What broke and what I learned

**A four-hour run died at its one indispensable stage.** The largest parallel build I have launched ran about four hours and 2.2 million tokens, then died at its single integrator, the one stage holding the test gate for everything upstream. Every parallel track had finished; the stage that made their work durable had not. That failure forced the redesign I still build with: every track gets a capped scope, integration splits into a foundation stage and a feature stage that owns the gate, and the next phase's review panel runs in parallel with the current build. The new shape got tested almost immediately, when another track died mid-run under API flakiness. This time the loss was one track's tail: the foundation stage's work survived, the parallel review panel delivered its findings in full, and I finished the last component by hand. A failure now costs a track, never the afternoon.

**Never trust a self-report.** During that recovery, a follow-up run made 22 tool calls over about 40 minutes and left zero durable changes on disk. Twenty-two tool calls read like steady progress; the repository said nothing had happened. Since then the rule is mechanical: after any failed run I run git status and re-run the test gate myself before believing anything landed. I treat a run's completion claims as unverified until the repository and the gate agree with them.

**A memory path that silently loaded nothing.** My scheduled jobs read their context from a knowledge base that lives on a network mount, and mounts drop. One loading path resolved a missing mount to an empty result and let the run continue as if it had context. The fix made loading fail loud: loaders now resolve a fallback chain (live mount, then a local replica), flag their output when it came from the fallback, and stop with a visible error when neither source is reachable. An automation that cannot see its inputs should stop and say so.

**Administering the server taught its own lessons.** The one I cite most: on TrueNAS datasets with restricted NFSv4 aclmode, chmod fails on every file even as root, and changing ownership to the container's uid is the entire fix. I learned that from a terminal full of "Operation not permitted", then wrote it down as a durable note that my future sessions load before touching container permissions. Same principle as the outcome ledger: a lesson that lives only in my head is a lesson I will pay for twice.

## The automation layer

The same machinery runs my week. A scheduled job writes a morning brief into that day's note at 6 a.m. Session handoffs are structured artifacts: ending a work session writes a continuation file plus a session note with frontmatter for decisions made, next actions, and duration, then links that note to its project and to the sessions before it. A new session starts by reading them, so context survives across days and machines instead of living inside one chat window. Automation output routes through a single notification utility with four channels (push, iMessage, email, or an append to the knowledge base) and rules about which channel fits which message; every scheduled task reports through it. The project layer reads the same knowledge base, currently 22 project folders, each carrying its own notes, tasks, and session history, so the dashboard and the scheduled jobs work from the same source of truth I do.

## Tools

- Next.js 16 (App Router), React 19, TypeScript, Tailwind 4
- SQLite via better-sqlite3: WAL mode, FTS5 full-text search, one file as the entire datastore
- React Flow with dagre auto-layout for the workflow DAG view, Recharts for trend charts
- cron-parser for schedules, chokidar for file watching, Server-Sent Events for live streams
- Google Calendar API with OAuth for two-way calendar sync
- Docker, TrueNAS, Syncthing, Tailscale (WireGuard)

Everything above has a written record: the schema, the formulas, the incident notes, and the design decisions. Ask me to walk through any of it: bryanwudarsky@gmail.com.
