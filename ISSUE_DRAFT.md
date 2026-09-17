# [DRAFT — not yet filed] Issue for PR #1253

**Title:** Analysts run sequentially, making the analyst phase unnecessarily slow

**Labels (suggested):** performance, enhancement

---

## Summary

The four analysts (Market, Sentiment, News, Fundamentals) run as a single
sequential chain, so the analyst phase takes as long as the **sum** of all four
even though the analysts are independent of one another. This is the dominant
latency component of a run.

## Current behavior

In `graph/setup.py`, analysts are wired back-to-back:

```
START → Market → (tools loop) → Msg Clear → Sentiment → … → Fundamentals → Bull Researcher
```

Each analyst only starts after the previous one's `Msg Clear` node fires. They
share a single `messages` channel, cleared between each analyst — which is the
only reason they cannot run concurrently.

## Why they can run in parallel

The analysts are independent:

- Each reads only the shared run inputs (ticker, date, resolved identity).
- Each writes exactly one `*_report` field and never reads another analyst's output.
- The `Msg Clear` step between them proves no cross-analyst message history is needed.

The single shared `messages` channel is used only as a per-analyst ReAct
scratchpad, so the sole blocker to concurrency is message isolation.

## Proposed change

- Wrap each analyst in its own compiled ReAct subgraph with a **private
  `messages` channel**, so analysts cannot clobber each other's tool scratchpad.
- Fan out from `START` to all selected analysts in a single superstep, and fan
  back in at the Bull Researcher (which runs once after all reports are in).
- Only each analyst's `*_report` field crosses back to the parent graph.
- Remove the now-unnecessary `Msg Clear` nodes (the subgraph `END` discards
  messages).

Downstream nodes (researchers, trader, risk team, portfolio manager) are
unchanged.

## Impact

Analyst wall-time drops from **sum → max** of the four analysts, with no change
to outputs.

## Scope / non-goals

- Intra-run analyst parallelism only. Cross-run / cross-ticker concurrency is
  out of scope.
- Behavior of all downstream stages is unchanged.

## Notes / trade-offs

- **Checkpoint resume:** a subgraph invocation is atomic to the parent
  checkpointer, so on a `--checkpoint` resume an unfinished analyst re-runs from
  scratch rather than mid-loop. Acceptable — analysts simply re-fetch.
- **CLI wall-time tracker** assumes sequential execution; under parallelism its
  per-analyst timings become approximate (not broken).

## Testing

- New unit tests covering message isolation and the fan-in barrier (no LLM
  required).
- Full existing suite passes with no regressions.

## References

- Implemented by #1253
