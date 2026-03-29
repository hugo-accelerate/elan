# Tool Comparison Plan

This note keeps candidate external examples and a shared baseline graph for side-by-side rewrites in Elan.

The goal is:

- compare syntax on the same workflow shape
- check whether Elan stays more readable as graph features accumulate
- surface missing features before implementation work starts

## Candidate Tools

- Airflow
  - dynamic task mapping
  - branching
- LangGraph
  - subgraphs
  - conditional routing
  - human-in-the-loop later
- Prefect
  - nested flows
  - mapped task patterns
- Dagster
  - graphs and ops
  - dynamic outputs and mapping
- Metaflow
  - branching
  - foreach and join
- CrewAI
  - flows
  - router/listener patterns

## Shared Baseline Graph

The first comparison set uses one graph shape across tools as much as possible.

```text
start
  -> load_items
  -> process_each(item)    fan-out
  -> collect_results       join / reduce
  -> decide
      -> notify            conditional branch
      -> store             conditional branch
  -> result
```

This graph covers:

- linear flow
- fan-out
- join / reduction
- conditional routing
- explicit result

It does not cover sub-workflows yet.

## Baseline Example Semantics

Use a simple numeric version first so the syntax stays the focus.

```text
load_items() -> [1, 2, 3]
process_each(x) -> x * 2
collect_results(values) -> sum(values)
decide(total) -> should_notify = total > 10
notify(total)
store(total)
result -> total
```

Expected output:

```text
[1, 2, 3]
-> [2, 4, 6]
-> 12
-> notify + store
-> result = 12
```

## Follow-Up Comparison Graphs

After the baseline graph, the next useful comparisons are:

1. Sub-workflow composition

```text
start
  -> child_workflow(sum_then_scale)
  -> result
```

2. Dynamic expansion

```text
start
  -> inspect
  -> dynamically append extra steps
  -> continue into existing static node
  -> result
```

3. Cyclic workflow

```text
start
  -> think
  -> decide
      -> act
      -> think
  -> result
```

4. Agent workflow

```text
start
  -> plan
  -> tool step(s)
  -> optional human approval
  -> result
```

## Comparison Order

1. Airflow baseline graph
2. Prefect baseline graph
3. Dagster baseline graph
4. Metaflow baseline graph
5. LangGraph baseline graph
6. Elan rewrite for the same graph

Then repeat for:

- sub-workflow composition
- dynamic expansion
- cyclic workflows
- agent-specific patterns

## Notes

- The first pass compares syntax, not runtime guarantees.
- If one tool cannot express the exact same graph directly, the nearest official idiom is acceptable.
- Human-in-the-loop and tool-calling belong in the later agent comparison set, not the baseline set.
