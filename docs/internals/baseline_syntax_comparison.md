# Baseline Syntax Comparison

This note compares one small workflow across several orchestration tools and Elan.

The goal is syntax comparison. It is not a runtime, deployment, or observability comparison.

## Baseline Graph

```text
start
  -> load_items
  -> process_each(item)
  -> collect_results
  -> decide
      -> notify            optional
      -> store             always
  -> result
```

Shared semantics:

- `load_items() -> [1, 2, 3]`
- `process_each(x) -> x * 2`
- `collect_results(values) -> sum(values)`
- `decide(total) -> total > 10`
- `notify(total)` runs only when the decision is true
- `store(total)` always runs
- `result -> total`

Expected outcome for this example:

```text
[1, 2, 3]
-> [2, 4, 6]
-> 12
-> notify + store
-> result = 12
```

## Airflow

Closest idiom:

- TaskFlow API
- dynamic task mapping with `expand()`
- task-id branching with `@task.branch`

```python
from datetime import datetime

from airflow.decorators import dag, task
from airflow.utils.trigger_rule import TriggerRule


@dag(start_date=datetime(2024, 1, 1), schedule=None, catchup=False)
def baseline():
    @task
    def load_items() -> list[int]:
        return [1, 2, 3]

    @task
    def process_each(x: int) -> int:
        return x * 2

    @task
    def collect_results(values: list[int]) -> int:
        return sum(values)

    @task.branch
    def decide(total: int) -> list[str]:
        if total > 10:
            return ["notify", "store"]
        return ["store"]

    @task
    def notify(total: int) -> None:
        print(f"notify {total}")

    @task
    def store(total: int) -> int:
        return total

    @task(trigger_rule=TriggerRule.NONE_FAILED_MIN_ONE_SUCCESS)
    def result(total: int) -> int:
        return total

    items = load_items()
    processed = process_each.expand(x=items)
    total = collect_results(processed)

    route = decide(total)
    notify_task = notify.override(task_id="notify")(total)
    store_task = store.override(task_id="store")(total)

    route >> [notify_task, store_task]
    [notify_task, store_task] >> result(total)


baseline()
```

Verdict:

Airflow makes the fan-out and reduce steps clear, but the branching model is task-id and skip based. The final `result` step needs a trigger rule because skipped branch paths affect downstream execution. This is close to the baseline graph, but more scheduler-shaped than graph-shaped.

## Prefect

Closest idiom:

- `@flow` and `@task`
- task mapping with `.map()`
- conditional control flow in plain Python

```python
from prefect import flow, task


@task
def load_items() -> list[int]:
    return [1, 2, 3]


@task
def process_each(x: int) -> int:
    return x * 2


@task
def collect_results(values: list[int]) -> int:
    return sum(values)


@task
def notify(total: int) -> None:
    print(f"notify {total}")


@task
def store(total: int) -> int:
    return total


@flow
def baseline() -> int:
    items = load_items()
    processed = process_each.map(items)
    total = collect_results(processed)

    if total > 10:
        notify.submit(total)

    stored = store.submit(total)
    return stored.result()
```

Verdict:

Prefect is compact because most control flow is ordinary Python. The tradeoff is that the branch is not expressed as workflow routing. The graph is implicit in the flow function, and the final result is just the flow return value.

## Dagster

Closest idiom:

- ops and graphs
- dynamic mapping with `DynamicOut` and `.map(...)`
- `.collect()` for the reduce step

```python
import dagster as dg


@dg.op(out=dg.DynamicOut())
def load_items():
    for value in [1, 2, 3]:
        yield dg.DynamicOutput(value, mapping_key=str(value))


@dg.op
def process_each(x: int) -> int:
    return x * 2


@dg.op
def collect_results(values: list[int]) -> int:
    return sum(values)


@dg.op
def notify_if_needed(total: int) -> None:
    if total > 10:
        print(f"notify {total}")


@dg.op
def store(total: int) -> int:
    return total


@dg.graph
def baseline_graph():
    processed = load_items().map(process_each)
    total = collect_results(processed.collect())
    notify_if_needed(total)
    store(total)


baseline_job = baseline_graph.to_job(name="baseline_job")
```

Verdict:

Dagster expresses the fan-out and collect steps directly. The weak spot for this graph is conditional routing in the same minimal style: the cleanest compact version moves the condition inside an op instead of expressing a branch in the graph itself. The result is a good map-reduce example, but not a clean one-to-one match for the routing part.

## Metaflow

Closest idiom:

- `FlowSpec`
- `foreach` for fan-out
- explicit `join` step

```python
from metaflow import FlowSpec, step


class BaselineFlow(FlowSpec):
    @step
    def start(self):
        self.items = [1, 2, 3]
        self.next(self.process_each, foreach="items")

    @step
    def process_each(self):
        self.processed = self.input * 2
        self.next(self.collect_results)

    @step
    def collect_results(self, inputs):
        self.total = sum(inp.processed for inp in inputs)
        self.next(self.decide)

    @step
    def decide(self):
        self.should_notify = self.total > 10
        self.next(self.notify, self.store)

    @step
    def notify(self):
        self.total = self.total
        if self.should_notify:
            print(f"notify {self.total}")
        self.next(self.result)

    @step
    def store(self):
        self.total = self.total
        self.next(self.result)

    @step
    def result(self, inputs):
        self.total = next(inp.total for inp in inputs)
        print(f"result = {self.total}")


if __name__ == "__main__":
    BaselineFlow()
```

Verdict:

Metaflow makes `foreach` and `join` explicit and readable. The branching model is broader than the baseline graph because both branch steps always exist and one branch becomes a no-op when notification is not needed. The result step is an explicit join, not a reserved output concept.

## LangGraph

Closest idiom:

- `StateGraph`
- `Send` for map-reduce fan-out
- conditional edges for routing

```python
from operator import add
from typing import Annotated
from typing_extensions import TypedDict

from langgraph.graph import END, START, StateGraph
from langgraph.types import Send


class State(TypedDict, total=False):
    items: list[int]
    item: int
    processed: Annotated[list[int], add]
    total: int
    should_notify: bool
    stored_total: int


def load_items(state: State):
    return {"items": [1, 2, 3]}


def fan_out(state: State):
    return [Send("process_each", {"item": item}) for item in state["items"]]


def process_each(state: State):
    return {"processed": [state["item"] * 2]}


def collect_results(state: State):
    return {"total": sum(state["processed"])}


def decide(state: State):
    return {"should_notify": state["total"] > 10}


def route_after_decide(state: State):
    if state["should_notify"]:
        return ["notify", "store"]
    return ["store"]


def notify(state: State):
    print(f"notify {state['total']}")


def store(state: State):
    return {"stored_total": state["total"]}


builder = StateGraph(State)
builder.add_node("load_items", load_items)
builder.add_node("process_each", process_each)
builder.add_node("collect_results", collect_results)
builder.add_node("decide", decide)
builder.add_node("notify", notify)
builder.add_node("store", store)

builder.add_edge(START, "load_items")
builder.add_conditional_edges("load_items", fan_out)
builder.add_edge("process_each", "collect_results")
builder.add_edge("collect_results", "decide")
builder.add_conditional_edges("decide", route_after_decide)
builder.add_edge("notify", END)
builder.add_edge("store", END)

graph = builder.compile()
```

Verdict:

LangGraph can represent the map-reduce and routing parts directly, but the syntax is state-and-edge centric. The workflow reads like a programmable graph runtime rather than a task workflow DSL. It is expressive, but more mechanical than the baseline Elan form.

## Elan

Closest idiom:

- `Workflow` and `Node`
- `yield`-based fan-out through ordinary node execution
- child workflow boundary for the map-reduce segment
- explicit reserved `result` node

```python
import elan as el
from pydantic import BaseModel
from elan import Join, Node, When, Workflow


@el.ref
class DecisionPayload(BaseModel):
    total: int
    should_notify: bool


@el.task
def decide(total: int) -> DecisionPayload:
    return DecisionPayload(total=total, should_notify=total > 10)


@el.task
def load_items():
    yield 1
    yield 2
    yield 3


@el.task
def process_each(item: int) -> int:
    return item * 2


@el.task
def collect_results(values: list[int]) -> int:
    return sum(values)


@el.task
def notify(total: int) -> None:
    print(f"notify {total}")


@el.task
def store(total: int) -> int:
    return total


map_reduce = Workflow(
    "map_reduce",
    start=Node(run=load_items, next="process_each"),
    process_each=Node(run=process_each, next="result"),
    result=Join(run=collect_results),
)


workflow = Workflow(
    "baseline",
    start=Node(run=map_reduce, next="decide"),
    decide=Node(
        run=decide,
        next=[
            When(DecisionPayload.should_notify, "notify"),
            "result",
        ],
    ),
    notify=Node(run=notify),
    result=Node(run=store),
)
```

Verdict:

Elan keeps the workflow centered on tasks, routing, and explicit graph structure instead of scheduler primitives or shared-state mechanics. The main friction this comparison exposes is the current `Join` restriction: a mid-graph reduce needs a child workflow boundary because `Join` is terminal-only on `result`. That keeps the first implementation narrow, but it is more indirect than the target baseline graph.

## Synthesis

Airflow, Dagster, and Metaflow express the fan-out and reduce parts clearly, but each carries more framework-specific machinery around the branch and final result.

Prefect is compact because it treats most orchestration as plain Python control flow. That keeps the code small, but the graph structure is less explicit.

LangGraph can model the full pattern, but it reads as a low-level graph runtime with shared state and edge functions rather than a task workflow DSL.

Elan keeps the task and routing vocabulary cleaner than the other tools:

- tasks stay as ordinary callables
- fan-out remains part of node execution
- routing stays explicit
- the workflow export stays explicit through `result`
- a mid-graph reduce currently needs a child workflow boundary

The main features still left to pressure-test in later comparisons are:

- sub-workflow composition
- dynamic expansion with `Expand(...)`
- cycles
- agent-specific control flow

## References

- Airflow dynamic task mapping: https://airflow.apache.org/docs/apache-airflow/stable/authoring-and-scheduling/dynamic-task-mapping.html
- Airflow branching: https://airflow.apache.org/docs/apache-airflow/2.10.3/core-concepts/dags.html
- Prefect flows: https://docs.prefect.io/v3/concepts/flows
- Prefect tasks: https://docs.prefect.io/v3/concepts/tasks
- Dagster dynamic graphs: https://legacy-versioned-docs.dagster.dagster-docs.io/concepts/ops-jobs-graphs/dynamic-graphs
- Metaflow basics: https://docs.metaflow.org/metaflow/basics
- LangGraph graph API: https://docs.langchain.com/oss/python/langgraph/graph-api
- LangGraph Send example: https://docs.langchain.com/oss/python/langgraph/use-graph-api
