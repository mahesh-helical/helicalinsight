# InstantBI Dashboard Agent

Plan-then-execute agent for multi-chart dashboards. A question is turned into a stored investigation plan. A later request with the same dashboard id runs that plan chart by chart, then writes a narrative and a layout.

| Layer | Location |
|-------|----------|
| Java proxy | `POST /ai/agent-dashboard` (`InstantBIController`) |
| InstantBI HTTP | `helicalbi/controller/agent_dashboard.py` → `POST /agent-dashboard` |
| Plan | `helicalbi/sql_agent/investigation.py` → `create_and_store_plan` |
| Execute | `helicalbi/sql_agent/dashboard_graph.py` → `run_dashboard_agent` |
| Modes | `helicalbi/sql_agent/modes.py` — `fast` / `balanced` / `research` |

`/interactive` is a different pipeline. Fast chat lives in `helicalbi/interactive/fast` (see [INTERACTIVE_FLOW.md](../../docs/INTERACTIVE_FLOW.md)). Think chat reuses this agent's plan graph with `flat_sql=True`.

## Two HTTP phases

`agent_dashboard()` does not run the tool loop on the first call.

| Phase | How to trigger | What runs | Response `phase` |
|-------|----------------|-----------|------------------|
| Plan | Any question that is not an execute command | `create_and_store_plan` | `plan` |
| Execute | Query text `execute plan`, `execute_plan`, `run plan`, or `run_plan`, or `input.action` / `input.phase` in `execute`, `execute_plan`, `run`, `run_plan` | Load the stored plan, then `run_dashboard_agent` | `execute` |

Execute requires a plan already saved for the same `dashboardid` and `dashboard_sequence_id`. Otherwise the handler raises: ask a question first, then send `execute plan`.

Required body fields: `input.inputString`, `input.model.file`, `input.model.dir`, `input.dashboardid`. `dashboard_sequence_id` (alias `dashboard_seq_id`) defaults to `"1"`. `input.mode` or `input.agent_mode` selects the token profile; otherwise `dashboard.default_mode` from config. Chart ceiling is `dashboard.max_sub_questions`, applied inside `resolve_mode_profile`.

Optional plan hints: `input.persona` / `input.user_persona`, `input.strategy` / `input.strategy_id`, plus role and profile from `resolve_role_profile`.

Plan response highlights: `persona`, `strategy` (id only), `strategy_detail`, `plan` (public keys only), `asked_questions` taken from planned chart questions, empty `dashboard` and `final_answer`, `message` telling the client to send `execute plan`.

Execute response highlights: `final_answer`, `dashboard`, `sub_questions` (including `report_model`), `investigation_steps`, `attempt_count`, `token_usage`, and the same `plan` / `strategy` / `persona`.

## Token modes

| Mode | Purpose | Charts | Loops | Overview chars | LLM synthesizer | Semantic reuse |
|------|---------|--------|-------|----------------|-----------------|----------------|
| `fast` | Low tokens / latency | ≤2 | ≤10 | 1200 | no (rule-based summary) | yes |
| `balanced` | Default | ≤5 | ≤24 | 3500 | yes | yes |
| `research` | Deeper picture | ≤8 | ≤40 | 8000 | yes | no |

`config_max_charts` (`dashboard.max_sub_questions`) lowers `max_charts` but does not raise it. Aliases: `quick`/`cheap` → fast, `default`/`normal` → balanced, `deep`/`thorough` → research.

Planner tool bind list (`planner_tools`), not the ToolNode registry:

| Mode | Bound tools |
|------|-------------|
| `fast` | `retrieve_semantic_model`, `generate_sql`, `execute_query`, `build_report`, `finish_dashboard` |
| `balanced` | fast set plus `retrieve_schema`, `analyze_result` |
| `research` | balanced set plus `validate_sql` |

`dashboard_tools()` always returns the full registry so a ToolNode can run any call. Only `research` binds `validate_sql` to the planner LLM. Fast and balanced also skip intent rephrase when topics are already seeded (`skip_intent_rephrase`). Fast skips viz polish (`skip_viz_polish`). History kept for the planner is 2 / 4 / 8 tool rounds.

LangGraph `recursion_limit` is `max_tool_loops * 3 + 8` (`DASHBOARD_NODES_PER_TOOL_LOOP` plus exit overhead). On execute, a stored plan sets the chart cap to the planned chart count when `flat_sql` is set; otherwise it is `min(profile cap, planned count)`.

## Dashboard graph

```text
START
  ├─ investigation_plan.charts → execute_plan → synthesizer → dashboard → END
  └─ no planned charts         → planner ⇄ tools → apply_patches
                                    └─ stop → synthesizer → dashboard → END
```

`route_start` sends the run to `execute_plan` when `has_planned_charts` is true (each chart has a non-empty `question`). That node is deterministic: semantic retrieve → generate SQL → execute with retry → build report → insight. It does not call the planner LLM. After the charts, control goes straight to the synthesizer.

The free-form loop is the fallback (tests, or a run with no chart specs). `gathering_should_stop` ends it when `is_complete` is set, the chart cap is reached (`charts_complete`), or `tool_loop_count >= max_tool_loops`. The planner itself returns `is_complete` on those same caps before another LLM call. `apply_tool_patches` folds each tool's `state_patch` into state and shrinks the tool messages.

Fast mode synthesizer builds a numbered text summary from `build_run_summary` and does not call the LLM. Balanced and research call `SYNTHESIZER_PROMPT`. `build_run_summary` is programmatic: `asked_questions`, `attempt_count` (`tool_loop_count`), and `investigation_steps` (`chart` vs `lookup` from `include_in_dashboard`).

`run_dashboard_agent` loads the model session, installs in-memory schema and semantic indexers under `dashboard:{thread_id}`, then invokes the graph. `SchemaIndexer()` is constructed with no `persist_directory`, so the default index is `InMemoryVectorIndex`. Chroma is used only when a persist directory is passed and `chromadb` imports.

SQL execution goes through `execute_sql_state` → `SqlExecutor.process_flow`. Result rows are capped at `DEFAULT_RESULT_ROW_CAP` (50).

## Strategy engine

The old monolithic strategy module is split. Callers (controllers, plan graph, tests) should import the stable names from `helicalbi.sql_agent.strategy_tree`. That module is a facade. Implementation lives in `helicalbi.sql_agent.strategy`. Paths, ids, and scoring weights live in `helicalbi.sql_agent.constants`.

| Module | Responsibility |
|--------|----------------|
| `strategy/catalog.py` | Read `strategies/dashboard_strategies.json` and `strategies/analytical_strategy.json`. LRU-cached. `get_strategy` falls back to `default_strategy`, then the first entry. |
| `strategy/selection.py` | Pick one strategy. |
| `strategy/analytical.py` | Rank analytical operations for a question. |
| `strategy/diagnostics.py` | Match model diagnostic capabilities, classify their `uses`, and lock Think scope. |
| `strategy/prompts.py` | Prompt blocks for a strategy, the analytical catalog, and matched diagnostics. |
| `strategy/runtime.py` | `public_strategy` (id only), `public_persona` (nested strategy objects stripped), `resolve_plan_runtime` (layout/color from the catalog), `charts_from_strategy`. |
| `strategy/text_utils.py` | `clean` / `compact` string helpers. |

JSON catalogs stay in `sql_agent/strategies/`. `clear_strategy_catalog_cache()` drops the dashboard-catalog LRU (tests and hot reload).

### How a strategy is chosen

`select_strategy(question, persona=, hint=, diagnostic_capabilities=)`:

1. Use `hint`, else `persona.strategy_hint` or `persona.strategy_id`, if that id exists in the catalog. Selection source is `hint`. Hyphens and spaces are normalized to underscores.
2. Otherwise classify intent from catalog keyword lists (`classify_intent`; fallback intent is `overview`) and walk `decision_tree` with persona name, intent, and hint. Selection source is `decision_tree`.
3. If capabilities were passed, attach the first capability whose name or applicable measure is mentioned in the question.

Retired layout ids are aliases, not separate templates: `so_what_contextual_anchoring` → `inverted_pyramid`, `intent_driven_layout` → `semantic_color_coding`.

`create_and_store_plan` calls `select_strategy` once up front, then again when the semantic indexer (or `cube_info_prepared`) exposes diagnostic capabilities. `attach_strategy` stores the chosen id on the persona. The public API returns the id via `public_strategy`, not the full catalog object. `plan_to_public_dict` keeps `persona`, `tier`, `strategy_id`, `template_id`, `domain`, `topics`, `original_question`, `rationale`, `charts`, `analytical_operations`, `query_analytical_strategies`.

The plan-context graph (`nodes/plan_graph.py`, entry `run_plan_context_graph`) is:

```text
load_semantic_context → select_domain_topics → enrich_topic_context
  → select_analytical_operations → draft_chart_questions → validate_questions
       └─ validation_feedback and repair_count ≤ PLAN_GRAPH_MAX_REPAIR (1) → draft again
```

`dashboard_node` hydrates layout and color with `resolve_plan_runtime`.

### Diagnostics

Diagnostic capabilities come from the semantic model (`diagnosticCapabilities` or `diagnostic_capabilities`).

| Function | When it matters |
|----------|-----------------|
| `match_diagnostic_capabilities` | Capabilities whose name or applicable measures match the question. |
| `resolve_diagnostic_uses` | Each `uses` entry classified as strategy, measure, dimension, or other. |
| `expand_decompose_measures` | Formula components pulled in for decompose-style uses. |
| `operations_for_diagnostic_uses` | Analytical operations required by those uses. If empty, `select_analytical_operations` scores the catalog instead (default max 5). |
| `diagnostic_field_scope` / `diagnostic_scope_lock` | Think prompts stay inside the configured strategies, measures, and dimensions. |
| `charts_within_diagnostic_scope` | Drop chart fields outside that scope. |
| `measure_trait_guidance` | Grain, additivity, and direction text. Also used by `sql/GetContextForSQL.py`. |

The plan-graph node runs the match / resolve / expand path only when `flat_sql` is true and capabilities are present (Think). A normal dashboard plan still passes capabilities into `select_strategy`, which only does the lighter name-or-measure match above.

Custom analytical operations on the model are merged in `catalog.custom_operations_from_source` / `merge_custom_operations`. `list_analytical_operations_for_api` is the projection served to utility-config consumers.

## Prompts

Prompt strings live under `helicalbi/sql_agent/prompt/` and are re-exported from `helicalbi.sql_agent.config` so `from helicalbi.sql_agent.config import TOOL_AGENT_PROMPT` still works. Numeric knobs are re-exported from `constants` the same way.

| Module | Contents |
|--------|----------|
| `prompt/agent_prompts.py` | `TOOL_AGENT_PROMPT`, `PLAN_EXECUTION_RULES`, orchestration and evaluation prompts |
| `prompt/plan_prompts.py` | Plan-context and Think flat-SQL planning rules |
| `prompt/sql_prompts.py` | Think flat-SQL generation and subquery rejection text |
| `prompt/synthesis_prompts.py` | `SYNTHESIZER_PROMPT`, Think final-answer prompt |
| `prompt/insight_prompts.py` | Step insight and success-execution prompts |
| `prompt/routing_prompts.py` | `AUTO_MODE_PROMPT` (interactive auto router) |

`TOOL_AGENT_PROMPT` is formatted in `bootstrap_planner_messages` with `mode_rules`, the original question, a truncated semantic overview, the investigation plan, `PLAN_EXECUTION_RULES` when a plan exists, collected findings, and the chart and loop caps. The planner does not write SQL itself. Lookup steps are `generate_sql → execute_query → analyze_result`. Chart steps are `generate_sql → execute_query → build_report`. Stop with `finish_dashboard`.

## Topic scoring

`planning/scoring.py` (`TopicScorer` / `score_topic`) ranks semantic-model topic rows for `select_domain_topics_node`. It is lexical, not an embedding. Weights in `constants.py`:

| Hit | Score |
|-----|-------|
| Exact topic name contained in the question | 5.0 |
| Exact domain name contained in the question | 3.0 |
| Component name contained in the question | 2.5 |
| Question token (length ≥ 3) found in the topic blob | 1.0 |
| Component token (length ≥ 4) shared with the question | 0.5 |

`plan_graph._score_topic` is a compatibility alias. Domain and topic caps by mode, from `investigation._topic_caps_for_mode`: fast 1 domain / 2 topics, balanced 2 / 3, research 2 / 4.

## Tool scenarios

Semantic retrieve (`tools/semantic.py`):

1. Return the cached payload when the same question was retrieved earlier in the session.
2. In fast and balanced, if the last hit was `sufficient` and topics exist, reuse it (`reused: true`) and skip another RAG call.
3. Otherwise call the semantic indexer. If `sufficient` is false, call schema RAG and set `used_metadata_fallback`.

`generate_sql` records the question on `asked_questions` and stores SQL on `generated_sql`. When `flat_sql` is set it appends `THINK_FLAT_SQL_GENERATION` and rejects subqueries. It does not execute.

`execute_query` validates with `validate_sql_against_catalog` before `execute_sql_state`. A hard validation error increments `sql_retry_count` and does not execute. An `Unknown column` error is a soft failure only when physical metadata is loaded; the engine error then feeds the retry. Any execute error also increments `sql_retry_count`; success resets it to 0. The planner prompt tells the model to call `generate_sql` again with that error. `execute_plan` uses `execute_sql_with_retries` instead of the planner.

`analyze_result` appends a finding with `include_in_dashboard: false` (lookup, not a tile). `build_report` appends a finding with `include_in_dashboard: true`. `finish_dashboard` sets `is_complete`.

Agent SQL generation (`instantbi_agent_graph.py`) is separate from `/interactive`:

```text
prepare
  ├─ seeded topics → seed_domain
  │     ├─ fast/balanced → sql_generator
  │     └─ research → intent rephrase → sql_generator
  └─ no topics → main_graph → sql_generator
```

## Key files

```text
helicalbi/
  controller/agent_dashboard.py          # plan vs execute HTTP
  sql_agent/
    README.md
    dashboard_graph.py                   # execute_plan | ReAct planner
    investigation.py                     # create_and_store_plan, execute detection
    instantbi_turn.py                    # model session, execute_sql_state
    instantbi_agent_graph.py             # per-facet SQL branch
    modes.py                             # fast | balanced | research
    constants.py                         # catalog paths, scores, loop budgets
    config.py                            # re-exports prompts + numeric knobs
    strategy_tree.py                     # public facade — import this
    strategy/                            # catalog, selection, diagnostics, prompts, runtime
    planning/scoring.py                  # lexical topic rank
    prompt/                              # prompt strings
    strategies/                          # dashboard_strategies.json, analytical_strategy.json
    nodes/planner.py
    nodes/execute_plan.py
    nodes/plan_graph.py
    nodes/apply_patches.py
    nodes/synthesizer.py
    nodes/dashboard.py
    nodes/run_summary.py
    tools/tools.py                       # dashboard_tools vs planner_tools
    database/schema_indexer.py           # in-memory unless persist_directory is set
    database/semantic_indexer.py
```
