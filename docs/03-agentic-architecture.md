# Multi-Agent Swarm Architecture for the Vaisravana Fleet

**Status:** Design / planning. No execution.
**Part of:** sera-bot-architecture
**Related:** 01-database-architecture.md, 04-honest-evaluation.md

---

## 0. Why a swarm, not a monolith

The current fleet already runs an *autonomous improvement loop* (the wave bot's
30-min cron that proposes + measures + commits one change per tick). That loop
is a single-agent process wearing three hats (researcher, implementer, evaluator)
sequentially. It works, but it has structural limits:

- **No adversarial check.** A change that looks good to the proposer gets committed. There is no independent skeptic forcing the evidence bar higher.
- **No parallelism.** Three lenses (skeptic / practitioner / architect) run one after another, so a full iteration costs 3x wall-clock.
- **Single LLM brain.** One model proposes and evaluates. Confirmation bias is unfiltered.
- **No cross-bot learning.** The main bot, wave bot, and alpha bot each learn in isolation. A lesson in alpha (e.g. "fee-aware EV gate is the highest-ROI change") never propagates to main automatically.

A **swarm** fixes this by giving each lens its own agent, running them in parallel against a shared evidence base (the unified DB from 01-database-architecture.md), and synthesizing through a coordinator that is itself constrained by the Sentinel rule (only ParameterSurface + additive gates may change without val's approval).

---

## 1. Agent roles

```
                          ┌─────────────────────────────┐
                          │   COORDINATOR (orchestrator) │
                          │   - owns the goal queue       │
                          │   - enforces Sentinel          │
                          │   - merges agent verdicts      │
                          │   - writes LOOP_STATUS.md      │
                          └───────────────┬───────────────┘
                                          │
        ┌──────────────────┬──────────────┼──────────────┬──────────────────┐
        │                  │              │              │                  │
  ┌─────▼─────┐     ┌──────▼──────┐ ┌─────▼─────┐  ┌──────▼──────┐  ┌──────▼─────┐
  │ SKEPTIC   │     │ PRACTITIONER│ │ ARCHITECT │  │ RESEARCHER  │  │ AUDITOR    │
  │ (red-team)│     │ (what works)│ │ (buildable)│  │ (signal mining)│ │ (DB integrity)│
  └─────┬─────┘     └──────┬──────┘ └─────┬─────┘  └──────┬──────┘  └──────┬─────┘
        │                  │              │              │                  │
        └──────────────────┴──────────────┴──────────────┴──────────────────┘
                                          │
                          ┌───────────────▼───────────────┐
                          │   SHARED EVIDENCE BASE          │
                          │   (unified trading DB +         │
                          │    alpha-agentic.db + KB)       │
                          └─────────────────────────────────┘
```

### 1.1 SKEPTIC (red-team quant PM)
- **Job:** kill the thesis with hard numbers. "This change looks good" is not enough — show the expectancy math, the fee drag, the sample size, the regime-dependence.
- **Tools:** read-only DB queries, metrics scripts, statistical tests.
- **Output:** `REJECT` / `INCONCLUSIVE` / `ACCEPT-WITH-CAVEATS` with a numeric justification. Must cite MIN_TRADES guard (n >= 20 closed trades before any KEEP verdict).
- **Bound:** cannot write code. Only votes.

### 1.2 PRACTITIONER (what is viable at OUR constraints)
- **Job:** given a hypothesis, say whether it is implementable at a $10 paper account on a non-co-lo VPS with REST-poll ~5s latency. Rejects "just add HFT" or "just run funding arbitrage" as out of reach.
- **Tools:** latency probes, fee calculators, min-notional checks.
- **Output:** feasibility verdict + the exact constraint that binds.

### 1.3 ARCHITECT (concrete buildable design)
- **Job:** turn an accepted hypothesis into a concrete, Sentinel-compliant change: which file, which function, which ParameterSurface key, which gate.
- **Tools:** code read/write (restricted to allowed surfaces), diff generation.
- **Output:** a patch + a test + a one-line Sentinel classification (ParameterSurface / additive-gate / ENGINE-CHANGE-REQUIRES-APPROVAL).

### 1.4 RESEARCHER (signal mining)
- **Job:** mine new signal sources the bots don't have yet (order flow, funding rate divergence, OI changes, liquidation prints, whale alerts, social sentiment). Propose only signals with a free, reliable, low-latency source on seria.
- **Tools:** web_search, Binance public endpoints, free APIs.
- **Output:** a ranked list of signal candidates with source + wiring + expected information gain. NO code that touches the engine without approval.

### 1.5 AUDITOR (DB integrity + honest metrics)
- **Job:** every proposed change must pass the 6-layer eval (L0 integrity ... L5 risk) from the alpha bot's evaluation framework. The auditor is the gate that blocks a commit if fee consistency, net==gross-fees, or ts-order fails.
- **Tools:** alpha-eval CLI, schema checks.
- **Output:** PASS / FAIL with the failing layer named.

---

## 2. Coordinator loop (the swarm tick)

```
every 30 min (cron) OR on-demand (val /loop):
  1. COORDINATOR reads LOOP_STATUS.md + goal queue
  2. COORDINATOR dispatches the current hypothesis to SKEPTIC, PRACTITIONER,
     ARCHITECT in PARALLEL (delegate_task, background)
  3. RESEARCHER runs continuously in its own thread, feeding candidate signals
     into the shared evidence base
  4. When the 3 parallel agents return:
       - if any votes REJECT with numeric justification → log ITER block as
         REJECT, do NOT deploy, advance to next hypothesis
       - if all ACCEPT → ARCHITECT emits patch, AUDITOR runs 6-layer eval
           - AUDITOR PASS → deploy (clean reset), measure window, SKEPTIC
             re-evaluates the measured result
               - SKEPTIC KEEP → commit + push (valarion creds)
               - SKEPTIC REVERT → rollback, log negative result
           - AUDITOR FAIL → block, log which layer failed
  5. COORDINATOR writes LOOP_STATUS.md (last metrics, current candidate,
     soak_n vs MIN_N, next_action)
  6. loop again
```

### 2.1 Sentinel enforcement inside the coordinator
The coordinator refuses to apply any patch classified `ENGINE-CHANGE` unless val
has explicitly approved it in chat. ParameterSurface + additive-gate patches
apply automatically (that is the existing autonomous mandate). This prevents the
historical clobber problem (wave-loop reverting main-bot paper layer).

### 2.2 Cross-bot learning
When alpha discovers a high-ROI additive gate (e.g. fee-aware EV), the
coordinator writes it to a shared `knowledge-base/transfer.md`. The main and wave
bots' next ticks read that file and can adopt the gate as an additive change
without re-discovering it. This is the key scaling lever — the fleet gets smarter
as a whole, not per-bot.

---

## 3. Hermes-to-Hermes communication

You asked specifically about "agent swarm or hermes-to-hermes conversations."
Two patterns are available on this machine:

### 3.1 delegate_task (parallel subagents)
The `delegate_task` tool spawns isolated agent contexts that run in the
background and return a consolidated summary. This is the primary swarm primitive — each agent in section 1 is a `delegate_task` call. They share context via the evidence base (DB + files), not via chat.

### 3.2 Continuous hermes-chat loop (the "never stop" path)
For a true never-ending research sprint, the wave bot's `scripts/wave_research_loop.sh` pattern applies: a wrapper runs `hermes chat --max-turns 150` with a per-iteration cap, auto-restarts the next iteration reading prior `/root/wave_research/*.md`, and stops only when a sentinel file is removed. This is the "loop until the end of time" mode.

**Recommendation:** use `delegate_task` for the per-tick swarm (fast, isolated, parallel) and reserve the hermes-chat loop for deep research sprints where val wants a multi-hour autonomous dive. The two compose: a cron tick can spawn delegate_task agents, and a research sprint can be launched from a tick.

---

## 4. Evidence base (shared state)

All agents read/write a single source of truth:

- **Unified trading DB** (see 01-database-architecture.md) — all bots' trades, positions, markers, PnL, fees in one schema.
- **alpha-agentic.db** — runs, iterations, gate_rejections, evaluations, decisions. The swarm's working memory.
- **knowledge-base/** — transferable lessons (transfer.md, fee-model.md, signal-candidates.md).
- **LOOP_STATUS.md** — the coordinator's persisted state across the 30-min gaps.

Without a shared evidence base, a swarm is just N isolated bots talking past each other. The DB is the conversation.

---

## 5. Failure modes and guards

| Failure mode | Guard |
|---|---|
| Confirmation bias (one brain) | SKEPTIC votes REJECT independently; coordinator needs consensus |
| Overfitting to a tiny window | MIN_TRADES=20 guard; INSUFFICIENT is never a pass |
| Engine clobber | Sentinel classification; ENGINE-CHANGE needs val approval |
| DB lock contention | WAL + busy_timeout(30000) on agentic DB; read replicas for metrics |
| Infinite loop / runaway cost | per-iteration turn cap; sentinel file to halt; cron 30m pacing |
| Agents editing the same file | coordinator serializes patches; git as the merge layer |

---

## 6. Phased rollout (execution waits for val's go-ahead)

- **Phase A (week 1):** stand up the unified DB (01-database-architecture.md). No bot logic change — just centralized analytics + the 6-layer eval against real data.
- **Phase B (week 2):** promote the wave bot's existing 30-min loop to the coordinator + SKEPTIC/PRACTITIONER/ARCHITECT triple (delegate_task).
- **Phase C (week 3):** add RESEARCHER signal mining; wire `transfer.md` cross-bot learning.
- **Phase D (week 4+):** optional hermes-chat deep sprints for strategy-class research (the "think different" path that needs human-in-the-loop review).

See 06-implementation-roadmap.md for the full sequence.
