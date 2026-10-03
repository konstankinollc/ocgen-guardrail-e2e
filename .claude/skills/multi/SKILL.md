---
name: multi
description: Run the agent team on one task
argument-hint: "[task]"
disable-model-invocation: true
---
Task: $ARGUMENTS

Delegate it to each subagent in turn and wait for each to finish:
- @explorer
- @implementer
- @reviewer

Then consolidate the answers, flag disagreements, and give a final decision.
