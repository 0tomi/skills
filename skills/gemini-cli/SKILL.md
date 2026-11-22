---
name: gemini-cli
description: Invoke Gemini models non-interactively through the installed CLI, check subscription quota, and consume agent responses.
---

# Gemini CLI

Use the installed `agy` CLI to invoke Gemini models non-interactively.

## Default invocation

Invoke Gemini with:

```bash
agy -p "<PROMPT>" \
  --model <MODEL> \
  --print-timeout 30m
```

The command waits for Gemini to complete, prints its final response to `stdout`, and exits.

Diagnostics and errors are written to `stderr`.

Use the normal text output by default. Do not request JSON output for ordinary Gemini invocations.

## Models

Use the latest available version of each family (check with `agy models`):

* **Gemini Hydrogen**, effort `high`: default.
* **Gemini Neon**: only if the user wants to delegate critical, heavy, or intelligence-demanding work.

The effort is part of the slug (`gemini-<VERSION>-<FAMILY>-<EFFORT>`). Do not add an `--effort` flag.

Do not invent model slugs.

If an explicitly selected model is unavailable or invalid, treat it as an execution failure rather than silently substituting another model.

## Quota failure handling

If a Gemini invocation later fails, refuses to execute, times out unexpectedly, or produces no usable response, verify subscription quota:

```bash
agy -p "/usage"
```

If `/usage` confirms exhausted Gemini quota:

* Stop using Gemini.
* Do not retry the failed request.
* Do not repeatedly probe Gemini.
* Treat Gemini as unavailable for the remainder of the current orchestration run.

If `/usage` reports remaining quota, do not classify the failure as quota exhaustion.

Handle it as a normal execution, model, permission, timeout, configuration, or service failure.

Temporary service or capacity failures must not disable Gemini unless `/usage` confirms exhausted quota.

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

Prefer independent invocations unless previous Gemini conversation context is useful or required.

## Timeout

Override the headless timeout with:

```bash
--print-timeout <DURATION>
```

Recommended default:

```bash
--print-timeout 30m
```

## Permissions

Headless execution cannot answer interactive permission prompts.

The CLI applies its configured permission rules during execution.

Automatically approve safe actions with:

```bash
--mode auto
```

Use `--dangerously-skip-permissions` only when the user explicitly asks for it. It auto-approves every tool permission request without prompting:

```bash
--dangerously-skip-permissions
```

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
  --model <LATEST_HYDROGEN_HIGH> \
  --print-timeout 30m
```

Before the first Gemini invocation, check `/usage`.

After successful invocations, continue without repeatedly checking quota.

If a later invocation fails or refuses to produce a usable result, check `/usage` again. If Gemini quota is exhausted, stop using Gemini for the remainder of the current orchestration run.
