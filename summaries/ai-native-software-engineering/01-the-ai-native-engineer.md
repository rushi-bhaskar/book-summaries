# Chapter 1 — The AI-Native Engineer
*AI-Native Software Engineering · ~10 min read*

## TL;DR

- AI is automating the *typing* part of software engineering, not the *engineering* part. The job is moving from "implementer" to "orchestrator": you define intent, set constraints, verify output, and own the result.
- "Vibe coding" is great for throwaway prototypes and dangerous in production. The fix isn't less AI — it's AI plus engineering discipline.
- Three ideas sit at the centre of everything in this book: **intent, constraints, verification**.
- Eight skills define the AI-native engineer; the single habit to start with is: *write down what you want, the constraints, and what "done" means before you prompt.*

## Key concepts

### 1. The vibe-coding trap
Vibe coding (Karpathy's term, Feb 2025): describe roughly what you want, accept the model's diffs without reading them, paste errors back, repeat. It feels magical for prototypes. In production it fails predictably, because it mixes up two things: *using AI to speed up implementation* (good) and *giving up engineering judgment* (bad). The author suggests calling the good version **vibe prototyping** — fine when the goal is learning, not shipping. The skill is knowing which mode you're in.

Three named failure modes:

- **Plausibility gap** — AI code *looks* production-grade (clean names, familiar patterns), so your eyes stop looking for bugs. In code you wrote, you know where the fragile bits are; in vibe-coded code you have no map.
- **Architectural drift** — each prompt solves the local task, but the pieces don't fit: inconsistent schemas, diverging error handling, duplicated auth logic. Works today, expensive tomorrow.
- **Missing context** — your system's real rules (validation lives in the service layer, soft deletes not hard deletes, errors go through localisation) exist nowhere in the code. The model can't respect what it can't see.

Mental model: treat the agent like a **capable junior who is new to your codebase**. Fast and competent, but you are the reviewer who carries the context.

### 2. Two engineers, one ticket (the chapter's core example)
Ticket: rate-limit the payment endpoint to 10 requests/min per user.

- **Engineer A** prompts in one line. Agent builds a sliding-window limiter **keyed on IP** with a new Redis client. Tests pass, merged. Three days later: a corporate customer behind a shared NAT gets blocked, while a fraudster on a rotating residential IP walks straight past the limit. The code was correct for the problem *as stated*; the problem was stated wrong.
- **Engineer B** spends ~10 minutes writing notes first: key on authenticated `user_id` (auth middleware already provides it), don't count idempotent retries with the same idempotency key, emit the team's standard rate-limit metric, use the approved cache wrapper, Redis is in the same AZ. The agent's output now fits the system. B reviews it against the notes, spots a gap (idempotency key present but the earlier request *failed*), updates the spec, reruns, ships.

Same tool, ~10 extra minutes, completely different outcome: one incident + rollback + a day of cleanup vs. a working feature. **The difference was the engineering, not the AI.**

### 3. What AI-native engineering is *not*

- Not about building AI products — it's for people building payments, pipelines, web apps, infra.
- Not about the tool of the month — the practices should outlive any specific IDE plugin.
- Not about removing humans — judgment matters *more* (what to build, is it right, what to do when it's wrong).
- Not necessarily "coding less" — some people write more because they take on bigger work.
- Not seniors-only — juniors who learn spec writing and verification early compound faster.

### 4. From implementer to orchestrator
When an agent can produce a CRUD endpoint in 30 seconds, *writing* it yourself is worth less. What's still yours: what it should do, its invariants, malformed inputs, load behaviour, and whether the design fits the system. An orchestrator **defines work precisely → assigns it → reviews against intent → integrates what passes → redirects what doesn't.**

Practical shifts:

- More time *before* the work (constraints, edge cases, definition of done).
- **Review replaces writing** as the main activity — e.g. for the rate limiter, check missing user ID, expired session, idempotency key, Redis down — not just "does request #11 get a 429".
- More communication (product ↔ design ↔ infra ↔ security) — this "overhead" is now high-value work.
- **Verification is designed up front**, not bolted on.
- Hand-writing still matters for performance-critical paths, subtle algorithms, novel problems and deep system-specific code.

### 5. Why the old workflow stops scaling
Two old assumptions break:

1. **Implementation speed is the bottleneck** — it isn't any more. The bottleneck moves *upstream* (requirement/spec clarity) and *downstream* (verification).
2. **You author every artifact** — now your authorship is expressed through the decisions that guided generation and what you accepted or rejected.

"Discover the design while coding" worked when *you* wrote the code, because you accumulated understanding. Agents don't — they restart at every context boundary, fill gaps with assumptions, and then build more on top of those assumptions, so late corrections are costly. Boehm's classic finding (defects found in operation can cost ~100× more than in requirements) gets *amplified* by AI: you can generate a lot of misaligned code very quickly.

Team-level angle: tribal knowledge in people's heads was always a problem; with agents it's acute, because every unwritten rule has to be re-explained on every task. Teams that scale **write context down**: rules files, spec templates, conventions docs.

### 6. The new centre of gravity: intent, constraints, verification
They reinforce each other: good constraints sharpen intent; rigorous verification exposes constraints you forgot to state.

- **Intent** = what must be true after the work, not the steps. *Weak:* "Add a user profile page." *Strong:* return name, avatar URL, member-since, 2 latest public posts; 404 if missing; hide email for unauthenticated callers; cache 60s using the existing key convention. Strong intent removes the model's degrees of freedom.
- **Constraints** = boundaries: technical (use existing auth lib, p99 < 200 ms, no new DB), business (all locales, v2 backward compatibility, no email leaks), quality (80% coverage on new code, security scan, naming conventions). **#1 failure mode: the agent breaks a constraint it never knew about** — tests pass, it ships, it blows up later. Writing constraints down is the highest-leverage fix.
- **Verification** = where your judgment replaces the model's confidence. Models hallucinate APIs, misread code, make locally sensible but globally wrong assumptions. Three layers: automated (tests, lint, types, CI) → structured review against intent/constraints → acceptance testing under realistic conditions.

These map to old ideas (requirements, NFRs, QA). What's new is how much they matter once code is cheap.

### 7. Human-in-the-loop (HITL) as a design principle
HITL = humans explicitly approve decisions above a certain level of consequence or novelty before the system proceeds. The agent proposes; the engineer decides. Three forms:

1. **Review the spec/plan before execution** — the highest-value checkpoint (e.g. plan modes in coding tools).
2. **Review output before merging** — not "does it compile" (CI does that) but "does it match intent and fit the system".
3. **Escalate high-risk decisions** — auth flows, production data migrations, new external dependencies get deliberate review; a well-tested utility gets a quick one.

Key insight: it's **calibrated automation, not maximum automation**. Skipping HITL "for speed" is backwards — a 10-minute spec review is far cheaper than hours of fixing a misaligned implementation.

### 8. What an AI-native engineer's day looks like

- **Before:** write first — behaviour, constraints, edge cases, acceptance criteria. Maintain persistent context (rules files, conventions) as *infrastructure* that lives next to the code.
- **During:** review against intent → constraints → existing system. When output is wrong, **diagnose why** (missing case? unclear constraint? missing context?) and fix the spec. Re-rolling the prompt and hoping is a random walk.
- **Throughout:** every commit is yours — the agent is not a co-author who shares accountability. Stay hands-on for high-stakes moments. Invest in your context system (month 3 is much easier than month 1). Use AI beyond code: docs, PR descriptions, design exploration, explaining legacy code.

## The eight-skill stack
| Skill | One-line meaning |
|---|---|
| Spec literacy | Write precise, agent-executable intent: behaviour, constraints, "done" (Ch. 5) |
| Context engineering | Output is bounded by context — choose the files, conventions, examples, constraints (Ch. 3) |
| Orchestration | Break work into scoped subtasks, know dependencies and where risk lives |
| Verification discipline | Review against specific criteria, not gut feel; agent-written tests can pass without testing what matters |
| Risk ownership | The meta-skill: decide what to delegate and how hard to review |
| Systems thinking | Does this belong in this service? Will the data model hold in 6 months? |
| Communication & translation | Mockups → behavioural specs, vague asks → testable criteria |
| Harness engineering | **Agent = model + harness.** Feed-forward (rules, architecture docs) + feedback (tests, linters, type checks, AI review) so the agent can self-correct |

Technical depth is still required — it's now the floor, not the differentiator.

## Perspective & trade-offs

- **Cost of not adapting is invisible at first.** Traditional and unstructured-AI engineers both ship; the gap shows in scope, quality and sustained throughput. An engineer who specifies, delegates and verifies well can cover the scope of 2–3 engineers.
- **Team > individual.** Handing everyone an AI subscription with no shared conventions gets only a fraction of the value; shared specs, rules files and verification standards compound.
- **Career: AI replaces tasks, not purpose.** Radiology is the precedent — AI reads scans, yet demand for radiologists grew. **Jevons paradox:** cheaper code → more software becomes worth building → more need for people who define, verify and own it. Engineering is drifting toward a *leadership* role, earlier in careers.
- **Honest loss:** if you love hands-on coding, this shift is a real adjustment — but the problems on the other side (is this design safe, which trade-off, what fails at scale) are the hard, interesting ones.
- **Juniors:** the market is genuinely harder, but the path is the same and matters more — pick harder problems, learn tools one layer deeper, do design work, read agent code until you could have written it (turns the agent into a tutor), and find a team where seniors pair with you *on the agent*.

## Apply it (this week)

1. **The 5-minute rule:** before any non-trivial prompt, write intent + constraints + definition of done. Compare the output to your usual cold prompt.
2. **Start a rules file** for one repo you work on (e.g. `CLAUDE.md` / rules file): validation layer, error handling pattern, approved libs (your Redis/cache wrapper, Kafka conventions), naming. Add to it every time an agent violates an unwritten rule.
3. **Review checklist for AI PRs:** intent match → constraint violations → edge cases (nulls, auth expiry, retries/idempotency, dependency down) → fit with existing patterns.

## Think about it

1. In your current microservices, which "invisible" constraints (things not obvious from the code) would an agent most likely break first — and where are they written down today?
2. Engineer A's limiter was keyed on IP. What's an equivalent "correct for the problem as stated, wrong for reality" bug you've seen, and what one line of spec would have prevented it?
