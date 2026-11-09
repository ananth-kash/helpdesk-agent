# helpdesk-agent

An IT service-desk agent that works through a ticket queue. It reads each ticket, looks up the requester and their software licenses, searches the knowledge base, and proposes a fix: assign a license, resolve with a KB-backed answer, or escalate. **Nothing is written to the ticketing system until a named human approves.** Approvers can also edit the reply before it goes out.

It runs out of the box on demo data. Point it at a free **ServiceNow** developer instance and it works on real tickets.

```
$ helpdesk-agent run --decider rules

── INC1001: Need Figma access  [Priya Nair (Design)]
   intent=license_request confidence=0.80
   will assign_license Figma  (User requested a seat)
   will resolve_ticket  (License assigned)
   reply: You've been given a Figma license. It can take up to 15 minutes to appear when you sign in.
   approve / edit reply / reject? [a/e/R] a
   your name: Ananth
...
| Ticket  | Summary                         | Intent          | Done                    | Policy blocks |
| INC1001 | Need Figma access               | license_request | assigned Figma, resolved | - |
| INC1003 | Tableau for quarterly reporting | license_request | escalated               | No free Tableau seats (2/2 used) |
| INC1005 | Email down for whole sales floor| incident        | escalated               | - |
| INC1006 | Figma please                    | license_request | escalated               | Figma is limited to Design, Engineering; Rahul Mehta is in Sales |
```

## How it works

```
 helpdesk-agent (LangGraph)                       it-helpdesk MCP server
 ─────────────────────────                        ──────────────────────
 plan     ── list_open_tickets ─────────────────►  read tools
 gather   ── get_ticket, get_user_profile,  ─────►   (ticket, user + licenses,
             list_license_pools, search_kb           catalog, KB vector search)
 decide   LLM → validated Decision JSON
          (repair on bad JSON, fall back to rules)
 guard    deterministic policy: seats, department,
          grounding, confidence, incidents
 approve  ⏸  interrupt — human approves / edits / rejects
 execute  ── assign_license, resolve_ticket, ────►  write tools
             add_work_note                           (need approver; license policy
 report   table + trace (latency, tokens)              re-checked on the server)
                                                    backend: demo SQLite | ServiceNow
```

| File | What it does |
|---|---|
| `mcp_server.py` | MCP server (Python SDK v2). It has 5 read-only tools, 3 write tools, a `kb://{id}` resource and a `triage_ticket` prompt. Writes require a named approver, and `assign_license` re-runs the license policy on the server, so the guardrails hold for any MCP client, not only this agent. |
| `kb.py` | Vector retrieval over KB articles: TF-IDF embeddings ranked by cosine similarity. It needs no model download. Swap in neural embeddings or pgvector behind the same `search()` method. |
| `deciders.py` | `LLMDecider` asks Claude for JSON matching the `Decision` schema and validates it with Pydantic. Errors go back to the model for up to N repairs. `RuleDecider` is a keyword baseline. |
| `policy.py` | Deterministic checks run before anything reaches a human. It checks the license is in the catalog, the department is allowed, seats are free and the user doesn't already hold it. It also checks that resolutions cite a retrieved KB article and that confidence is at least 0.7, and it always escalates incidents. Blocked actions become an escalation with the reason. |
| `graph.py` | LangGraph planner-executor with an `interrupt()` approval step and a checkpointer. If a write is refused, the ticket is escalated with the error instead of being silently skipped. |
| `backends/` | `DemoBackend` (SQLite, seeded) and `ServiceNowBackend` (REST Table API: `incident`, `sys_user`, `kb_knowledge`, `alm_license`, `alm_entitlement_user`). |
| `tracing.py` | Records latency, LLM calls, tokens and errors for each node in `trace.jsonl`. |

## Run it

```bash
pip install -e ".[dev]"

helpdesk-agent run --decider rules         # no API key needed
export ANTHROPIC_API_KEY=...               # optional: HELPDESK_MODEL=claude-sonnet-5-5
helpdesk-agent run                         # Claude decides, you approve
helpdesk-agent run --approver "Ananth"     # batch mode, pre-approved by a named person
helpdesk-agent reset                       # re-seed the demo data

pytest -q
```

### Use your own ServiceNow

1. Get a free Personal Developer Instance at developer.servicenow.com.
2. Set the environment variables:
   ```bash
   export HELPDESK_BACKEND=servicenow
   export SERVICENOW_INSTANCE=dev12345        # your instance name
   export SERVICENOW_USER=admin SERVICENOW_PASSWORD=...
   helpdesk-agent run
   ```

Tickets, users and knowledge articles work on a stock instance. License assignment needs the Software Asset Management tables (`alm_license`, `alm_entitlement_user`). Without them, the agent escalates license requests instead of assigning seats.

### Use the MCP server from Claude Code, Cursor or Claude Desktop

```json
{ "mcpServers": { "it-helpdesk": {
    "command": "python", "args": ["-m", "helpdesk_agent.mcp_server"],
    "env": { "HELPDESK_DB": "/abs/path/helpdesk_demo.db" } } } }
```

## Evaluation

There are 24 labelled tickets in `evals/golden_set.jsonl`, covering license requests that should be granted or blocked, how-tos, access issues, incidents and out-of-scope requests.

```bash
python evals/run_eval.py --decider rules
python evals/run_eval.py --decider llm --baseline evals/results/rules.json
```

| Metric | What it checks |
|---|---|
| Intent accuracy | Predicted intent matches the label |
| Action accuracy | The final actions after policy match the label (the end-to-end outcome) |
| Unsafe proposals | How often the decider proposed something policy had to block |
| Grounded resolutions | KB-answered resolutions cite an article that retrieval returned |

**Rule baseline: 79.2% intent and 79.2% action accuracy.** Its misses show why an LLM is worth the cost:
- It assigned **Acrobat** to someone who asked for **Photoshop**, because both start with "Adobe". Policy can't catch a wrong-but-allowed product, so only evals catch this.
- It resolved "lost phone, no other signed-in device" with the self-service MFA article, which needs a trusted device. The right call is to escalate.
- It called a single-printer how-to an incident because the ticket mentions a "floor".

Run the LLM decider with `--baseline` to see whether it beats this on accuracy, and what it costs in tokens and latency.

## Tests

The 11 pytest tests cover:
- KB retrieval
- the MCP surface (tools, resources and prompts)
- approver enforcement and server-side license policy
- the policy rules
- structured-output repair, using a scripted fake model
- end-to-end runs for approve, edit and reject
- falling back to rules when the LLM fails
- the ServiceNow adapter's Table API requests, using a mocked HTTP transport
