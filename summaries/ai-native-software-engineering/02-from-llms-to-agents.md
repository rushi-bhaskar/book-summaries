# Chapter 2 — From LLMs to Agents
*AI-Native Software Engineering · ~10 min read*

## TL;DR

- An LLM is a statistical next-token predictor over compressed training patterns. Fluency, reasoning, coding ability and tool use all come from that one mechanism, conditioned on whatever is in the context window.
- **In-weights knowledge** is frozen and can be wrong. **In-context knowledge** is authoritative because you control it. When the model is wrong about your API, the fix is to put the right facts in context (grounding), not to hope for a better model.
- An **agent = LLM + tools + context + a loop**. Under every framework it is a `while` loop that calls the model, runs the tool calls it asks for, appends the results, and repeats.
- The design concerns people underrate: **tool descriptions**, **stopping conditions**, **context rot**, and **where humans review** (at artifact boundaries: plan, code, tests, diff).
- The models and tools will keep changing. What lasts is your **harness**: rules files, specs, tests, MCP configs.

## Key concepts

### 1. Tokens and the context window
Models read **tokens**, not words. In English prose that's roughly 3–4 characters per token. Code, symbols, emoji and CJK scripts cost more. A call like `getUserById(userId)` is about 7 tokens, and a 600-line TypeScript file can be 4–5k. So a 128k window fills up fast once you add the system prompt, several files, history and docs. Tokenizers differ by provider, so measure with the tokenizer of the model you actually call. You pay per token on every call.

The context window is the model's **working memory**. Anything outside it doesn't exist for the model. Bigger windows don't mean even attention. The "Lost in the Middle" research found **U-shaped attention**: content at the start and end is used reliably, and content in the middle much less. Newer models handle simple needle-in-a-haystack lookups well across their full window, but harder work (multi-hop reasoning, reconciling scattered facts) still degrades well before the limit. Quick win: **put acceptance criteria and the relevant code first**, not after paragraphs of background.

### 2. How generation actually happens
The Transformer's **attention** lets every token look at every other token in context. That's how a constraint in the system prompt can shape the last line of generated code. At inference time the model only predicts the next token, appends it, and repeats. There is **no global planner and no revision pass**, so an early mistake carries through everything after it. That's why long outputs drift.

### 3. Why models are good at code (and how they were trained)
Code is heavily represented in training data (GitHub, Stack Overflow, docs). It's strictly structured, so the training signal is clear, and it's full of repeated boilerplate (CRUD, test setup, config). Source code and its docs often appear in the same training run, which helps the model link the two.

Post-training shapes the raw model:
- **SFT**: fine-tuning on curated input/output pairs.
- **RLHF**: humans rank outputs, a reward model learns their preferences, and the LLM is tuned to score well. This makes models helpful and conversational, and it's also why they can be **confidently wrong**, since a polished wrong answer can outscore an awkward correct one.
- **RLVR**: reward comes from a deterministic **verifier** (did the code pass the tests?). GRPO, popularised by DeepSeek-R1, is the best-known algorithm. RLVR is a big reason models are better at code and maths than at factual recall. It also points to the book's main practice: **write the tests and acceptance criteria first, then use them as the signal.**

### 4. Reasoning models and test-time compute
Chain-of-thought helps because each reasoning token conditions the next, so the model uses its own output as scratch memory. **Reasoning models** are trained to do this internally. Their "thinking tokens" are ordinary tokens, and usually **billed even when hidden**. More thinking means more compute, which costs latency and money (OpenAI's `reasoning_effort`, Anthropic's extended-thinking budget).

Rule of thumb: use a standard model for routine, well-specified, quickly verifiable work (a one-line function, reformatting JSON). Use a reasoning model for tasks with many interacting constraints: architectural trade-offs, a subtle race condition, a schema with tricky invariants, or when a standard model's answer is *structurally* wrong rather than slightly off.

### 5. Nondeterminism: temperature, top-p, top-k
The model **samples** from a probability distribution. Temperature near 0 makes it close to greedy. Even then, GPU parallelism and batching mean runs can differ slightly, so "temp 0" isn't truly reproducible. Values of 0.7–1.0 are common defaults. 1.5 and up gets wild. Top-p keeps the smallest set of tokens whose combined probability passes a threshold. Top-k keeps the k most likely tokens.

Consequence: **you can't test LLM output by comparing strings.** Test behaviour instead: does it parse as JSON, does it handle the edge case, do the tests pass. Use low temperature for codegen, extraction and classification. Use higher temperature for brainstorming designs or generating varied edge-case tests.

### 6. The model landscape (decide by constraint, not hype)
Frontier families as of early 2026: GPT, Claude (Haiku/Sonnet/Opus), Gemini, plus strong open-weights models (Kimi, GLM, DeepSeek, Mistral, Qwen). The useful distinction is *openness*:

| Tier | What you get | Why pick it |
|---|---|---|
| Fully open | Weights + data + training code | Auditability, reproducibility; lower capability ceiling |
| Open-weights | Weights only (Llama, Mistral, Qwen, DeepSeek) | Self-host for privacy, latency, cost at scale |
| Closed | API access (GPT, Claude, Gemini) | Highest ceiling; VPC-style options exist (Claude on Bedrock, Azure OpenAI), but no bare-metal/on-prem |

Choose using four hard constraints: can data leave your cloud, do you need fine-tuning, can your team run models in-house, and how much vendor dependency you can accept. Fine-tuning bakes behaviour into the weights but needs data, compute and versioning. As a coding-agent user, you'll mostly just prompt.

Also: **SLMs** (under ~10B params: Phi, Gemma, small Llama/Qwen) can match big models on narrow tasks like ticket classification, structured extraction or boilerplate, at a fraction of the cost. Decide with an eval set, not gut feel. **Multimodal** input is great for screenshots of errors, UIs and hand-drawn architecture diagrams. It's weak at spatial precision (counting, pixel positions, dense charts) and will still answer confidently.

### 7. Embeddings, semantic search, grounding
An embedding model turns text into a fixed-length vector, and similar meanings land close together. "auth service returned 401" sits near "access denied, token expired" even with no shared words. Uses: RAG, semantic code search, and de-duplicating tickets or log messages. **Grounding** means injecting authoritative info into context before generation. RAG is the systematic version.

### 8. Where LLMs fail (calibrate, don't trust)
Good at: pattern synthesis, first drafts in mainstream stacks, explaining unfamiliar code, format translation (SQL dialects, API schema → TS types), applying many constraints consistently.

Failure modes that are easy to miss:
- **Arithmetic and counting**: use a code tool.
- **Hallucination**: invented methods that follow a real library's naming style. A better model won't fix this. Grounding and verification will.
- **Strict factual recall** about versions and specs: fluent and subtly wrong.
- **Multi-step logic**: a confident answer with a flawed middle step.
- **Post-cutoff events**: new releases, CVEs, API changes.
- **Self-calibration**: no signal for "I'm guessing here".
- **Prompt injection**: instructions hidden in a web page or doc look the same to the model as yours.

Easy failures (bad arithmetic) are caught quickly. The dangerous ones look right: a plausible fake method, a subtle bug in security code. Review those categories harder.

### 9. Benchmarks: a tier signal, not a decision
HumanEval, SWE-bench, LiveCodeBench, MMLU, GPQA. Goodhart's law applies: once a score is a target, labs optimise for it. When SWE-bench+ removed tasks whose fix was already written in the issue, solve rates dropped a lot. Add saturation (top models within noise of each other) and data contamination. Use benchmarks to shortlist. **The only benchmark that matters is your own tasks.**

### 10. The API layer
Most providers follow the OpenAI-style **messages array** with `system`, `user` and `assistant` roles, so switching providers is mostly config. The key fact is that **LLMs are stateless**: no server-side session, so your app resends the full history on every call. The **system prompt** is your highest-leverage input. It's present on every turn and biases everything, so "You are a code reviewer, focus on security and performance" changes what "review this function" returns. Other features: **structured outputs** (JSON-schema-constrained), **streaming**, and **function calling**. Because history is resent every time, a long session's cost and latency keep growing. That's why agents reset or compact context.

### 11. What makes an agent
Definition: **an LLM with tools, context, and a loop that pursues a goal over multiple steps without a human at each step.**
- **LLM**: the decision engine. Model quality matters most here, especially for staying on track across many steps.
- **Tools**: functions with a name, description and parameter schema. The model only *declares* a call; your code *executes* it (consultant vs contractor).
- **Context**: everything sent on this call. It's the only thing that makes stateless calls feel stateful.
- **Loop**: reason → act → observe → check if done → repeat. Also called Plan-Execute-Reflect or Think-Do-See. Same structure.

Multi-agent setups (an orchestrator plus subagents) are built from the same loop. Most tasks don't need them (Ch. 9).

### 12. An agent in ~50 lines
Based on Hugging Face's "Tiny Agents" post: *once you have an MCP client, an agent is just a while loop.* The pieces are an inference client, tools as JSON schemas (the chapter shows a `read_file` tool with a `path` parameter), and this loop:

```
messages = [user(goal)]
while True:
    resp = llm.generate(messages, tools)
    if not resp.tool_calls: return resp.content
    messages.append(resp)
    for call in resp.tool_calls:
        messages.append(tool_result(call.id, execute(call)))
```

OpenAI uses a `tool` role. Anthropic uses `tool_use`/`tool_result` content blocks. The control flow is the same. Tiny Agents adds `task_complete` and `ask_question` tools so the agent's exit is **explicit**.

**Stopping conditions** are required: max iterations (around 30 means it's stuck, so surface progress and ask), a context budget (summarise or reset), a wall-clock timeout, and ask-before-continuing (the cleanest, since a stall becomes a checkpoint). Frameworks add registration, memory, retries and observability, at the cost of abstraction. Start with the raw API and add a framework when you need one.

Why it matters even if you only *use* Claude Code, Cursor or Copilot: rereading a file means it's regrounding; stopping mid-task points to an iteration cap; misusing your MCP tool means check the description first; getting worse over a long session means the history is bloated.

### 13. ReAct and tests as observations
**ReAct** (Yao et al., 2022) interleaves *thought → action → observation*. The chapter's example: replacing deprecated `getUserById` with `fetchUser`. The agent searches the codebase (3 files), reads `src/api/users.ts`, rewrites line 23, and so on. Reasoning traces give you **interpretability**, **error localisation** (the bad assumption is visible at step 2) and **grounding** (it reads the real file instead of guessing).

The **self-correction loop**: the agent runs `npm test -- --testPathPattern=users`, sees "expected 404, got 200 when user not found", works out that a null check is missing, fixes it, and gets 4 passing tests. **How informative the failure message is limits how fast the agent converges.** "Expected 200, got 500" gives it little to go on. "Expected 200 matching UserSchema, got 500 `{error: missing field email}`" tells it what to fix. Linters and type checkers work the same way. So write tests first. TDD and agents fit together naturally.

### 14. Tool design is API design
A good tool has an action-style **name** (`run_tests`, not `helper`), a **description** (the most important part: what it does, when to use it, *when not to*, e.g. "not for binary files or files >1MB") and a **parameter schema** with described fields. Anthropic's SWE-bench team spent more time on tools than on prompts, and requiring **absolute paths** removed a whole class of errors where the agent `cd`'d into a subdirectory and got paths wrong. Write descriptions for a literal-minded junior choosing between similar-sounding tools. Because the model only declares calls and your code runs them, **the gap between declaration and execution is where human approval belongs**.

### 15. Memory, and context rot
- **In-context**: the history. Fast, but limited by window size and lost-in-the-middle.
- **Files**: `spec.md`, `plan.md`, `tasks.md`. They survive resets, can be versioned, and leave an audit trail. To make an agent remember a decision, have it write the decision to a file.
- **RAG**: a vector DB (Pinecone, OpenSearch, Weaviate, **pgvector** for Postgres). A good production default is hybrid: **BM25 + dense vectors + a reranker**. Chunking and reranking usually matter more than which vector store you pick. Coding agents like Claude Code and Codex CLI mostly **grep and read files** instead, because code paths are exact. Vectors work better for docs, tickets and runbooks. Similarity isn't relevance, so sanity-check what comes back.
- **Agent-written notes**: a daily handoff log the agent reads at the start of the next session.

**Context rot** is stale attempts, superseded decisions and contradictions piling up in a long session. Signs: repeated mistakes, breaking a constraint it agreed to earlier, losing track of the goal. Fixes: one session per task, reset between tasks, keep state in files, compact *before* the window fills, and **re-state key constraints at the start of each session**.

### 16. Agent economics
Each iteration is an LLM call, so a 15-step session costs about 15× a single call, *plus* the growing resent history. It takes 30 seconds to several minutes, versus 1–3 seconds for one call. A well-specified task usually finishes in 5–15 iterations, while a vague one can go past 50, so **spec precision directly reduces iterations and cost**. Keep interactive tasks tight and save long runs for background work. Nondeterminism also compounds across iterations: Monday's success can fail on Wednesday with no code change. Validate against acceptance criteria, not output.

### 17. Human-in-the-loop at artifact boundaries
Two runs of "build a new endpoint". **Unsupervised**: it compiles and passes its own tests, but the auth, error format and logging all differ from the team's conventions. **With a plan step**: two minutes of review adds "use our `authMiddleware`, the error schema in `src/errors/types.ts`, and the logger from `src/lib/logger.ts`", and it fits the first time. **Feedback is steering, and each correction becomes context.**

Review at **artifact boundaries** (plan → code → test results → final diff), not every tool call. Better still is **human on demand**: tell the agent "if requirements are ambiguous or you're unsure which approach matches our conventions, ask rather than guess." Set the level of oversight by risk and **reversibility**: a Markdown doc gets reviewed at the end, a production SQL migration before it runs, and an auth middleware change at every stage.

### 18. The tool landscape and the harness
The categories last longer than the products: **IDE assistants** (Copilot, Cursor, Zed, JetBrains), **CLI agents** (Claude Code, Codex CLI, Aider, Gemini CLI; scriptable and CI-friendly, but no inline review), **cloud/background agents** (Copilot cloud agent, Cursor background agents, Devin, Codex cloud; delegate and get a PR back, for well-specified work), **review bots** (CodeRabbit, Greptile, Graphite, Copilot review; a first pass, not judgment) and **UI generators** (v0, Lovable, Bolt, Figma Make, Claude Design; for exploration, don't ship their output directly).

How to choose: try it on 2–3 real backlog tasks, then check context control, latency/cost, privacy, auditability and **portability**. An **agent = model + harness**. Part of the harness is inside the product (context assembly, compaction, tool routing), which is why "best model" and "best agent" are different questions. The rest is yours (rules files, specs, skills, tests, MCP servers) and it moves with you between tools.

## Perspective & trade-offs

- **Understanding beats prompt tips.** If you know it's next-token prediction over a context window with uneven attention, you can work out most "tricks" yourself: put important content first, ground the facts, test behaviour.
- **Bigger isn't automatically better.** Reasoning models, large windows and big parameter counts all cost latency and money. An SLM or a standard model on a well-scoped task often wins. Measure on your own work.
- **Frameworks vs raw loops.** Frameworks save boilerplate but make debugging harder. The author's view: learn the raw loop, then add abstraction you've confirmed you need.
- **RAG isn't always the answer for code.** For exact symbol lookup, grep plus file reads is deterministic and cheaper. Use vectors for unstructured knowledge.
- **Autonomy is a dial.** Too much review wastes the agent's speed; too little lets it fill every gap with guesses. Set it by how reversible the action is.
- **Security is built in, not added later.** Prompt injection follows from how attention works. Any agent reading untrusted content (web pages, issues, emails) needs execution gates.

## Apply it (this week)

1. **Build the loop yourself** (the chapter's exercise): in TypeScript or Python, call any OpenAI-compatible API with 2–3 tools (`get_current_time`, `add_numbers`, or something real like `redis_get`/`pg_query` against a local Docker stack). Add a max-iterations cap and an `ask_question` tool. Then watch Claude Code or Cursor and identify the same parts.
2. **Make test failures informative**: in one Node/Next.js service, change assertions so failures show *expected shape vs actual body* (e.g. zod schema diffs), not just status codes. Give the agent that test command as its feedback loop.
3. **Audit one tool or MCP description** you use (a Kafka publish or DB migration tool, say): add when *not* to use it, require absolute paths/IDs, and put an approval gate before anything irreversible (prod migrations, topic deletes, `kubectl apply` to prod).

## Think about it

1. Which facts in your stack (internal Kafka topic conventions, your Redis key scheme, recent library upgrades) is an agent most likely getting from stale in-weights knowledge, and where would you inject the correct version?
2. In your last long agent session, did you see context rot (repeated mistakes, forgotten constraints)? What would you have moved into a file and re-stated at the start of a fresh session?
