# Diagnostic capabilities

Authors attach diagnostic capabilities to a semantic model so Think mode can stay on the measures, dimensions, and analytical strategies they configured. The editor is the diagnostic capabilities panel. Matching and scope locking live in `helicalbi/sql_agent/strategy/diagnostics.py`.

The panel is `DiagnosticCapabilitiesPanel` in `client/src/components/hi-agent/components/semantic-metadata-editor/agent-shelves/diagnostic-capabilities-panel.jsx`. `agent-workspace.jsx` writes the result onto cube field data, which is saved with the semantic model.

## Saved shape

Two lists travel with the model. Both camelCase and snake_case names are accepted when the model is indexed (`diagnosticCapabilities` / `diagnostic_capabilities`, `customAnalyticalOperations` / `custom_analytical_operations`).

```json
{
  "diagnosticCapabilities": [
    {
      "capability": "Margin decline",
      "applicableTo": [{ "measure": "Gross Margin" }],
      "uses": [
        {
          "analyticalOperation": "decompose",
          "measure": "Gross Margin",
          "dimension": ["Region", "Product"]
        }
      ]
    }
  ],
  "customAnalyticalOperations": [
    {
      "operation": "custom strategy 1",
      "type": "custom",
      "category": "custom",
      "description": "",
      "requiredInputs": ["measure"],
      "optionalInputs": [],
      "execution": { "method": "sql", "logic": "" },
      "custom": true
    }
  ]
}
```

| Editor field | Persisted as | Read by |
|---|---|---|
| Capability name | `capability` | `match_diagnostic_capabilities` (name hit) |
| Applicable measures | `applicableTo: [{ "measure": "..." }]`. A bare string or `name` / `fields` is also accepted. | Same matcher, plus the semantic overview |
| Strategy row | `uses[].analyticalOperation`. `operation`, `strategy`, and `analytical_operation` are accepted on read. | `resolve_diagnostic_uses` |
| Required / optional slots | Other keys on the use object. One bound name stays a string; several become a list. Empty slots are omitted. | Input bindings on a strategy use |
| Custom strategy | `customAnalyticalOperations[]` with `type: "custom"` | Merged in front of the JSON catalog; a custom name replaces a catalog operation with the same name |

`requiredInputs` and `optionalInputs` on a **use** are editor-only. They come from the operation catalog so the panel can show drop slots. `toPayload` does not write those two keys. Input values are stored under the slot name (`measure`, `dimension`, and any extra key the catalog declares).

The operation catalog is loaded from `ai/utility/analytical-operations` (`operations[]` with `operation`, `requiredInputs`, `optionalInputs`). The panel shows an error and an empty catalog when that call returns `status: 0` or fails. Custom strategies are still editable.

## Editor constraints

- A capability cannot list the same analytical operation twice.
- Applicable measures and input values are unique, case-insensitive.
- Dropped cube fields contribute `fields`, then `alias`, `columnName`, `name`, or `text`.
- `agent-workspace.jsx` keeps a draft and an echo guard. Cube field updates clone `diagnosticCapabilities`; a clone of the list from **before** the edit must not replace the capability the user just added. Opening another model (its field keys change) replaces the draft, including an empty list.
- Save reads `cubeFieldsData.diagnosticCapabilities`. The workspace dispatches `setCubeFieldsData` on each edit so the pane and the save payload stay the same list.

## When Think uses them

`SemanticIndexer.index_model` stores both lists. `load_semantic_context_node` copies them into plan-graph state. The full diagnostic path runs only inside `select_analytical_operations_node`, and only when `flat_sql` is true and at least one capability is present. Think mode sets `flat_sql`. A Fast question does not build a diagnostic use plan; capabilities there only boost analytical-operation scoring.

Order when `flat_sql` is on:

1. `match_diagnostic_capabilities` keeps a capability when an applicable measure is in the question (or in its synonyms) or the capability name itself is mentioned. Each hit is copied with `matchedMeasures`.
2. `resolve_diagnostic_uses` classifies every use as `strategy`, `measure`, `dimension`, or `other`.
3. If the plan contains a `decompose` strategy, `expand_decompose_measures` adds measure names found inside those measures’ formulas (`source: "formula"`). Formula names shorter than 3 characters are ignored.
4. Grain, additivity, and direction from measure metadata are copied onto the use plan and turned into prompt notes (`measure_trait_guidance`).
5. `diagnostic_field_scope` is the allowed strategy / measure / dimension set. `diagnostic_scope_lock` is injected into the planner prompt when at least one strategy was resolved. `charts_within_diagnostic_scope` then runs only when that scope has strategies (`build_investigation_plan` calls it when `flat_sql` and `scope["strategies"]`). A chart is dropped when its `operation` / `analytical_strategy` is outside the set, when any `measure_hints` entry is outside the measure set (or the hints are empty and no allowed measure appears in the question), or when any `components` entry is outside the measure+dimension set (or no allowed dimension appears in the question or components). If no strategy was resolved, charts are left unchanged. An empty kept list is replaced by one chart per configured strategy (`_charts_for_configured_scope`). With diagnostic capabilities and `flat_sql`, the plan is also capped at 5 charts.

Mention rules (`MeasureMentionDetector`):

| Case | Counts as a mention |
|---|---|
| Name or synonym, length ≥ 3 | Substring of the question |
| One distinctive token | Token length ≥ 6 and it appears in the question |
| One long token inside a multi-word name | Token length ≥ 8 |
| Otherwise | At least two tokens, exact or one edit away |
| Generic tokens | `rate`, `cost`, `total`, `count`, `value`, `amount`, `average`, `percent`, and the other tokens in `GENERIC_MEASURE_TOKENS` are ignored |

A capability with an empty `applicableTo` matches only when its name is mentioned.

Use classification:

- An explicit `type` / `kind` of `strategy`, `operation`, or `analytical_strategy` is a strategy. The name is canonicalized against the catalog.
- A key containing `dimension`, or the key `entityidentifier`, binds dimensions. `benchmark`, `startingvalue`, `endingvalue`, `changecomponents`, or a key containing `measure`, binds measures.
- With no hint, a name that matches a catalog operation (including aliases) is a strategy. Otherwise a model measure or dimension name wins. Anything left is `other` and does not lock scope.
- Aliases include `measuredecomposition` → `decompose`, `periodcomparison` → `period_comparison`, `segmentanalysis` → `segment`, `trendanalysis` → `trend`.

Trait notes the planner can see:

| Trait | Effect |
|---|---|
| `grain` | Stay at that grain |
| `non additive` / `not additive` | Do not SUM; compute from formula components inside each group |
| `semi additive` | Do not SUM across time |
| `additive` | SUM is valid across dimensions at this grain |
| `direction` `negative` / `positive` / `neutral` | Whether an increase is unfavorable, a decrease is unfavorable, or the change is neither |

## Pitfalls

- Editing the panel without a successful model save leaves Think on the previous `diagnosticCapabilities`.
- A strategy name that is not in the catalog and not an alias becomes `other`. It will not appear in the scope lock, so the planner is not forced onto it.
- Scope locking applies only after a strategy use resolves. Measures alone narrow grounding context and `allowed_names`; they do not drop charts.
- A chart that names an allowed strategy can still be dropped when its measure hints or components include a field outside the resolved scope, or when the question text does not mention an allowed measure or dimension.
- Custom strategies must keep `type: "custom"` (or `custom: true`). Rows without that marker are dropped by `isCustomStrategy` and by `custom_operations_from_source`.
