---
name: gemini-cli
description: Invoke Gemini models non-interactively through the installed CLI, check subscription quota, and consume agent responses.
---

# Gemini CLI

Use the installed `agy` CLI to invoke Gemini models non-interactively.

## Initial quota check

Before the first Gemini invocation of the current orchestration run, check subscription quota:

```bash
agy -p "/usage" --output-format json
```

Perform this check once before using Gemini for the first time.

Inspect the `Gemini Models` quota group.

If either required quota bucket has no remaining quota:

* Do not invoke Gemini.
* Do not retry Gemini.
* Treat Gemini as unavailable for the remainder of the current orchestration run.

If quota remains available, Gemini may be used normally.

Do not run `/usage` before every successful invocation.

## Default invocation

Invoke Gemini with:

```bash
agy -p "<PROMPT>" \
  --model <MODEL> \
  --print-timeout 10m
```

The command waits for Gemini to complete, prints its final response to `stdout`, and exits.

Diagnostics and errors are written to `stderr`.

Use the normal text output by default. Do not request JSON output for ordinary Gemini invocations.

## Models

Use one of these Gemini model slugs:

```text
gemini-3.7-flash-high
gemini-3.1-pro-high
```

These model variants already select high reasoning effort. Do not add an `--effort` flag.

Inspect currently available models with:

```bash
agy models
```

Do not invent model slugs.

If an explicitly selected model is unavailable or invalid, treat it as an execution failure rather than silently substituting another model.

## Quota failure handling

If a Gemini invocation later fails, refuses to execute, times out unexpectedly, or produces no usable response, verify subscription quota:

```bash
agy -p "/usage" --output-format json
```

If `/usage` confirms exhausted Gemini quota:

* Stop using Gemini.
* Do not retry the failed request.
* Do not repeatedly probe Gemini.
* Treat Gemini as unavailable for the remainder of the current orchestration run.

If `/usage` reports remaining quota, do not classify the failure as quota exhaustion.

Handle it as a normal execution, model, permission, timeout, configuration, or service failure.

Temporary service or capacity failures must not disable Gemini unless `/usage` confirms exhausted quota.

## Streaming

Use streaming only when intermediate execution supervision is useful:

```bash
agy -p "<PROMPT>" \
  --model <MODEL> \
  --output-format stream-json \
  --print-timeout 10m
```

`stream-json` emits newline-delimited JSON events.

Relevant events include:

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

For ordinary Gemini invocations, prefer the default text output instead.

## Agent selection

List configured agents with:

```bash
agy agents
```

Select a specific configured agent with:

```bash
--agent <AGENT_NAME>
```

Example:

```bash
agy -p "<PROMPT>" \
  --agent <AGENT_NAME> \
  --model <MODEL> \
  --print-timeout 10m
```

Use `--agent` only when a particular configured Gemini agent is required.

## Conversations

Continue the latest conversation with:

```bash
agy -p "<PROMPT>" --continue
```

or:

```bash
agy -p "<PROMPT>" -c
```

Resume a known conversation with:

```bash
agy -p "<PROMPT>" \
  --conversation <CONVERSATION_ID>
```

Prefer independent invocations unless previous Gemini conversation context is specifically required.

## Timeout

Override the headless timeout with:

```bash
--print-timeout <DURATION>
```

Recommended default:

```bash
--print-timeout 10m
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

## Execution failures

A hard failure may produce:

* a non-zero process exit code
* diagnostic output on `stderr`
* an empty or unusable response
* a timeout

When a failure could plausibly be caused by exhausted subscription quota, run `/usage` before deciding whether Gemini remains available.

## Default command

```bash
agy -p "<PROMPT>" \
  --model gemini-3.7-flash-high \
  --print-timeout 10m
```

Before the first Gemini invocation, check `/usage`.

After successful invocations, continue without repeatedly checking quota.

If a later invocation fails or refuses to produce a usable result, check `/usage` again. If Gemini quota is exhausted, stop using Gemini for the remainder of the current orchestration run.
