---
name: gemini-cli
description: Invoke Gemini models non-interactively through the installed CLI, check quota, and consume agent results.
user-invocable: true
---

# Gemini Headless

Use the installed `agy` CLI to invoke Gemini models non-interactively.

## Initial quota check

Before the first Gemini invocation of the current orchestration run, check the available subscription quota:

```bash
agy -p "/usage" --output-format json
```

Perform this check once before using Gemini for the first time.

If `/usage` reports that the intended Gemini model has no remaining quota:

* Do not invoke that model.
* Do not retry it.
* Treat Gemini as unavailable for the remainder of the current orchestration run.

If quota is available, Gemini may be used normally.

Do not run `/usage` before every successful invocation.

## Basic invocation

```bash
agy -p "<PROMPT>"
```

`-p`, `--print`, and `--prompt` are equivalent.

The command executes the prompt, prints the Gemini agent result to `stdout`, and exits.

Diagnostics and errors are written to `stderr`.

## Default invocation

Prefer machine-readable JSON:

```bash
agy -p "<PROMPT>" \
  --model <MODEL> \
  --effort high \
  --output-format json \
  --print-timeout 10m
```

Always use:

```text
--effort high
```

Never use `low` or `medium`.

## Models

Use one of these Gemini model slugs:

```text
gemini-3.7-flash-high
gemini-3.1-pro-high
```

Inspect currently available models with:

```bash
agy models
```

Do not invent model slugs.

If an explicitly selected model is unavailable or invalid, treat that as an execution failure rather than silently substituting another model.

## JSON output

Use:

```bash
--output-format json
```

A result has this general structure:

```json
{
  "conversation_id": "...",
  "status": "SUCCESS",
  "response": "...",
  "duration_seconds": 0,
  "num_turns": 1,
  "usage": {
    "input_tokens": 0,
    "output_tokens": 0,
    "thinking_tokens": 0,
    "cache_read_tokens": 0,
    "total_tokens": 0
  }
}
```

The final textual result is available in:

```text
response
```

It may be extracted with:

```bash
agy -p "<PROMPT>" \
  --model <MODEL> \
  --effort high \
  --output-format json \
  | jq -r '.response'
```

## Quota failure handling

If a Gemini invocation later fails, refuses to execute, or returns no usable response, do not immediately assume the subscription quota is exhausted.

First verify quota again:

```bash
agy -p "/usage" --output-format json
```

If `/usage` confirms that the relevant Gemini quota is exhausted:

* Stop using Gemini.
* Do not retry the failed request.
* Do not probe Gemini repeatedly.
* Treat Gemini as unavailable for the remainder of the current orchestration run.

If `/usage` shows remaining quota, the failure is not considered quota exhaustion. Handle it as a normal execution, model, permission, timeout, or service failure.

Temporary service errors or capacity errors must not permanently disable Gemini unless `/usage` confirms exhausted quota.

## Streaming

When intermediate execution information is required:

```bash
agy -p "<PROMPT>" \
  --model <MODEL> \
  --effort high \
  --output-format stream-json
```

`stream-json` emits newline-delimited JSON events.

Relevant event types include:

```text
init
step_update
result
```

`step_update` may expose:

* response fragments
* tool executions
* tool results
* token usage
* subagent activity

The final `result` event contains the terminal result.

Prefer normal `json` output when intermediate supervision is unnecessary.

## Agent selection

List configured agents with:

```bash
agy agents
```

Select one with:

```bash
--agent <AGENT_NAME>
```

Command structure:

```bash
agy -p "<PROMPT>" \
  --agent <AGENT_NAME> \
  --model <MODEL> \
  --effort high \
  --output-format json
```

## Conversations

Executions start without previous conversation context unless explicitly resumed.

Continue the latest conversation:

```bash
agy -p "<PROMPT>" --continue
```

Equivalent shorthand:

```bash
agy -p "<PROMPT>" -c
```

Resume a specific conversation:

```bash
agy -p "<PROMPT>" \
  --conversation <CONVERSATION_ID>
```

`conversation_id` is available in JSON results.

When explicitly selecting a model, continue using `--effort high`.

## Timeout

Override the headless timeout with:

```bash
--print-timeout <DURATION>
```

Recommended default:

```bash
--print-timeout 10m
```

## Structured output

Constrain the final result with a JSON Schema when required:

```bash
--json-schema '<JSON_SCHEMA>'
```

When combined with:

```bash
--output-format json
```

the parsed result is available through:

```text
structured_output
```

## Permissions

Headless execution cannot answer interactive permission prompts.

The CLI applies its configured permission rules during execution.

Enable terminal sandbox restrictions with:

```bash
--sandbox
```

Automatically approve permission requests with:

```bash
--dangerously-skip-permissions
```

Do not use `--dangerously-skip-permissions` unless unrestricted tool execution is explicitly intended.

## Status and errors

For JSON output, inspect:

```text
status
```

Possible statuses include:

```text
SUCCESS
ERROR
CANCELED
INTERRUPTED
INVALID
WAITING
RUNNING
```

Failure details may appear in:

```text
error
```

Hard execution failures may also produce a non-zero process exit code and diagnostic output on `stderr`.

A failed Gemini invocation must trigger the quota verification procedure above when quota exhaustion is a plausible cause.

## Default command

```bash
agy -p "<PROMPT>" \
  --model <MODEL> \
  --effort high \
  --output-format json \
  --print-timeout 10m
```

Before the first Gemini invocation, check `/usage`.

After successful invocations, continue normally without repeatedly checking quota.

If a later Gemini invocation fails or refuses to produce a usable result, check `/usage` again. If quota is exhausted, stop using Gemini for the remainder of the current orchestration run.
