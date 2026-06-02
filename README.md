# Enterprise Agent Patterns — Failure Modes & Recovery

*Initial scaffolding — content coming.*
# Enterprise Agent Patterns — Failure Modes & Recovery

Working notes on the seven agent patterns from Anthropic's 
*Building Effective Agents*, framed around what actually breaks 
in production and how to recover. Drawn from shipping agents for an SMB financial intelligence 
platform with a live MCP server) and 20+ years building production systems in regulated finance industry.

These are opinions, not pure summary. Goal: discuss any of these 
in a senior interview without re-reading the essay.

---

## The meta-distinction: workflows vs. agents

**Workflow:** Predefined sequence of LLM and tool calls — 
the developer wires the path.

**Agent:** The LLM decides the path at runtime — choosing tools, 
iterating, deciding when to stop.

**When to choose workflow over agent:**
- Path is predictable and auditability matters 
  (regulated industries, financial close, claims processing)
- Cost per run must be bounded 
  (workflows are deterministic and cheap to model)
- You need to debug specific failure points 
  (graph structure makes this trivial vs. open-ended agent loops)

**When agent is worth the complexity:**
- Task space is open-ended (research, code editing, browsing)
- LLM's planning genuinely adds value vs. fixed branching
- You can afford the non-determinism and cost variance

**Senior judgment:** Default to workflows. Reach for agents 
when a workflow would need 50+ branches you'd rather let 
the LLM navigate.

---

## 1. Prompt Chaining

**What it is:** Sequential LLM calls — output of step N feeds 
step N+1. Optional verification gates between steps.

**When to use:** Tasks decomposable into verifiable linear 
sub-tasks (research → draft → review → format).

**Failure modes:**
- **Error cascade:** Mistake in step 2 corrupts steps 3-5 
  silently — wrong answer with high confidence
- **Cost stacking:** 5 chained calls × $0.05 = $0.25/run; 
  at 10K runs/day that's $2,500/day
- **Latency stack:** Each step adds 2-3s; 5 steps = 
  10-15s user wait with no feedback

**Recovery:**
- Add Pydantic validation gate between each step; 
  reject + retry with error feedback before proceeding
- Cache deterministic intermediate results in Redis; 
  skip on retry (idempotent step design)
- Stream step 1 output to user while step 2 runs 
  in parallel — cuts perceived latency in half

---

## 2. Routing

**What it is:** A classifier LLM (or rule) directs each 
input to one of N specialized downstream handlers.

**When to use:** When input types need genuinely different 
handling and merging them into one prompt hurts quality 
(e.g., billing vs. technical vs. refund queries).

**Failure modes:**
- **Misclassification sends to wrong handler:** 
  A refund query routed to technical support; 
  customer gets wrong answer confidently
- **New input type hits no route:** 
  Falls to a default handler that wasn't designed for it
- **Classifier drift:** Fine-tuned or prompted classifier 
  degrades over time as input distribution shifts

**Recovery:**
- Always have a catch-all "uncertain" route that 
  escalates to human rather than guessing
- Log every routing decision with confidence score; 
  alert when confidence drops below threshold
- Treat routing decisions as an eval target — 
  build a labeled test set of 50+ examples 
  and run regression on every deploy

---

## 3. Parallelization

**What it is:** Same task run N times in parallel and 
aggregated (voting/ensembling) — or different sub-tasks 
run in parallel and combined (sectioning).

**When to use:** Latency-bound work where N concurrent 
calls is faster than 1 sequential; or where multiple 
perspectives improve quality (e.g., security review 
from 3 different threat models simultaneously).

**Failure modes:**
- **Cost explosion:** 10 parallel calls per user request; 
  at scale this is 10x your expected API spend
- **Aggregation failure:** Majority vote works poorly 
  when all N models share the same bias
- **Partial failure handling:** 2 of 5 calls fail; 
  do you aggregate the 3, retry all 5, or error out?

**Recovery:**
- Hard cap on parallelism (max_concurrent) with 
  cost gate before fan-out
- Design aggregation to degrade gracefully: 
  if N-2 calls succeed, aggregate those; 
  log and alert on the failures
- Budget-based parallelism: calculate cost of 
  N parallel calls before starting; 
  abort if over threshold

---

## 4. Orchestrator-Workers

**What it is:** An orchestrator LLM decomposes a task 
into sub-tasks, dispatches each to worker LLMs (or tools), 
then synthesizes the results.

**When to use:** Tasks where sub-task structure can't be 
predicted in advance (open-ended research, multi-file 
code changes, complex data pipelines).

**Failure modes:**
- **Fan-out cost blowup:** Orchestrator spawns 50 workers 
  because it misclassified the task complexity; 
  single request costs $5
- **Worker result poisoning:** One worker returns 
  hallucinated data; orchestrator synthesizes it 
  with equal weight to correct results
- **Orchestrator gets stuck:** Workers complete but 
  orchestrator loops re-delegating instead of synthesizing

**Recovery:**
- max_workers cap + per-task cost gate before fan-out; 
  orchestrator must estimate task count before starting
- Workers return confidence scores alongside results; 
  orchestrator down-weights low-confidence outputs
- Max iteration limit on orchestrator loop + 
  fallback: "I have partial results, here's what 
  I have so far" rather than infinite retry

---

## 5. Evaluator-Optimizer

**What it is:** One LLM produces output, a second evaluates 
it, the first revises based on feedback. Loop until quality 
threshold is met or max iterations reached.

**When to use:** Tasks with clear evaluation criteria where 
iteration genuinely improves quality (translation, code 
review, structured report writing).

**Failure modes:**
- **Infinite loop:** Evaluator never satisfied; 
  agent iterates 20+ times burning $10+ per request
- **Evaluator drift:** Evaluator LLM is itself wrong 
  or inconsistent — negative feedback on good output, 
  positive on bad output
- **Convergence theater:** Agent makes cosmetic changes 
  each round to satisfy evaluator without real improvement

**Recovery:**
- Hard max_iterations (3-5 is usually right); 
  return best attempt at cap with a "max iterations 
  reached" flag rather than erroring out
- Evaluator should output structured scores 
  (Pydantic) not just pass/fail — 
  track scores across iterations; 
  stop if delta < threshold
- Build a golden eval set — run evaluator against 
  known-good outputs and known-bad outputs; 
  fail the deploy if evaluator accuracy drops below 80%

---

## 6. Autonomous Agents

**What it is:** LLM in a tool-use loop with environment 
access, deciding its own actions until goal is met 
or it gives up.

**When to use:** Open-ended tasks where the path genuinely 
can't be predetermined (coding assistants, research, 
browser automation). Trust and cost ceiling are high.

**Failure modes:**
- **Runaway iteration:** Agent loops indefinitely 
  on a blocking sub-task; cost unbounded
- **Hallucinated tool calls:** Agent calls tools 
  with parameters it invented; 
  downstream system accepts them silently
- **Prompt injection from environment:** 
  A webpage or file the agent reads contains 
  instructions that hijack its behavior
- **Stuck in local optima:** Agent keeps retrying 
  the same failing approach instead of changing strategy

**Recovery:**
- Max iterations + cost gate per session (hard limits, 
  not soft warnings)
- Tool schemas with strict Pydantic validation; 
  tools should return structured errors the agent 
  can reason about, not silent failures
- Treat all environment content as untrusted; 
  scrub tool outputs before feeding back to LLM; 
  use a separate "is this an injection?" classifier 
  on high-risk inputs
- Detect stuck loops: if last 3 tool calls are 
  identical, force a "change strategy" prompt 
  or escalate to human

---

## 7. Augmented LLM (the foundation)

**What it is:** A single LLM call equipped with retrieval, 
tools, and memory. The primitive everything else is built on.

**Why it matters:** Every pattern above is composed from 
this. If your augmented LLM is unreliable, no orchestration 
layer fixes it. Fix the foundation first.

**Failure modes:**
- **Context window overflow:** Retrieved docs + 
  conversation history + tools > context limit; 
  LLM silently truncates and loses critical information
- **Tool result ignored:** LLM has the right tool 
  result but reasons around it instead of using it
- **Memory pollution:** Long-running conversation 
  accumulates irrelevant context that degrades quality

**Recovery:**
- Track token count explicitly before each call; 
  summarize or evict old context when approaching limit
- Structured tool outputs (JSON) + system prompt 
  instruction to always reference tool results 
  before answering; eval specifically for tool usage
- Separate short-term (in-context) from long-term 
  (vector store) memory; only retrieve what's 
  relevant to current turn

---

## What I tell engineers picking a pattern

1. **Start with the simplest pattern that could work.** 
   Most production tasks need prompt chaining or routing, 
   not autonomous agents.

2. **Measure before you orchestrate.** A well-evaluated 
   single LLM call beats a poorly-evaluated 
   multi-agent system every time.

3. **Cost-cap every loop.** Any pattern with iteration 
   needs a hard ceiling on calls and dollars per run. 
   No exceptions.

4. **Audit every action.** In regulated contexts, 
   the audit log is the product — 
   without it, you can't ship.

5. **Fail loudly, not silently.** Silent failures 
   (wrong answer with high confidence) are worse 
   than loud failures (error with clear cause). 
   Design every pattern to surface problems 
   rather than paper over them.

---

*Last updated: May 31, 2026*  
*Anil Kandi*
