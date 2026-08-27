---
description: "Use when creating, editing, refactoring, or reviewing Kamailio configuration files such as kamailio.cfg. Apply the CaptionCall Kamailio style guide."
name: "Kamailio Style Guide"
applyTo: ["**/kamailio.cfg", "**/kamailio*.cfg", "**/*.kamailio.cfg"]
---

# Kamailio Configuration Style Rules

Apply these rules when modifying existing Kamailio configuration files. Preserve
nearby style when it is more specific, and avoid formatting churn outside the
lines needed for the task.

## File Layout

Keep files organized in this order unless a dependency requires otherwise:

1. Shebang and preprocessor options.
2. Global parameters.
3. Module loading.
4. Module parameters.
5. Flags, constants, and reusable definitions.
6. Main request route.
7. Named routes.
8. Branch, failure, reply, onreply, and event routes.

Use short banner headings only for major sections, such as global parameters,
modules, and routing logic.

## Naming

- Use upper-case names with underscores for `define`, `subst`, `defenv`, and
  related preprocessor values.
- Prefer `!!` for preprocessor lines after the `#!KAMAILIO` shebang.
- Use meaningful flag and constant prefixes: `FLT_`, `FLB_`, `AVP_`, and
  `HDR_`.
- Use upper-case route names that describe the route's action or decision.
- Name `$var()` and `$avp()` values after the data they hold, not after the
  storage mechanism.
- Use `$var()` only for local scratch values in the same synchronous route path.
- Use `$avp()` for SIP message or transaction state that may be used across
  branches, failures, replies, or on-reply routes.

## Whitespace and Indentation

- Use four spaces for indentation and no tabs.
- Keep braces on the same line as route declarations and control statements.
- Put `else` and `else if` on a new line after the previous block's closing
  brace.
- Do not leave a blank line immediately after a route declaration.
- Put two blank lines between route blocks.
- Put one blank line between logical blocks within a route.
- Use inner whitespace for condition parentheses: `if ( $var(i) == 0 )`.
- Do not use inner whitespace for function-call parentheses:
  `xinfo("message\n")`.

## Module Parameters

- Keep module parameters near the corresponding `loadmodule` section when
  practical.
- Group `modparam` lines by module.
- Do not interleave `modparam` lines for different modules.
- Put one blank line between each module's `modparam` block.
- Prefer named preprocessor constants for environment-specific parameter
  values.

Good:

```kamailio
loadmodule "tm.so"
loadmodule "sl.so"
loadmodule "rr.so"

modparam("tm", "failure_reply_mode", 3)
modparam("tm", "fr_timer", 30000)

modparam("rr", "enable_full_lr", 1)
modparam("rr", "append_fromtag", 1)
```

## Line Length

Try to keep lines to 80 characters. Break long strings with adjacent string
literals when the command supports it. For multi-argument function calls, break
after commas and align continuation lines under the first argument.

For long boolean expressions, put each major condition on its own line. Put the
boolean operator at the beginning of each continuation line, align the
operators, and place the closing parenthesis and opening brace on their own
line aligned with the `if`.

## Comments and Logging

- Use `##` for normal comments.
- Reserve banner comments for major sections.
- Use `/* ... */` sparingly for short multi-line notes.
- Comments should explain why, not restate the next line.
- Prefer `xinfo()`, `xwarn()`, and `xerr()` over `xlog()` with explicit levels.
- Include useful troubleshooting context such as `$ci`, `$rm`, `$ru`, `$si`,
  `$sp`, and route names.
- End log messages with `\n`.
- Do not log secrets, credentials, tokens, or full payloads unless there is a
  specific approved debugging need.

## Control Flow and Routes

- Keep route control flow direct and explicit.
- Use guard clauses and early exits for preconditions.
- Keep `request_route` high level.
- Prefer named routes for repeated logic.
- Avoid deeply nested `if` blocks.
- Make every `drop`, `exit`, `return`, and `sl_send_reply()` easy to justify
  from nearby context.
- Structure named routes as preconditions, local setup, main behavior, then
  exit or return behavior.

## SQL

When using `sqlops`, use stable connection names, log enough context to identify
the SIP transaction and SQL operation, handle lookup misses separately from hard
failures when possible, assign query strings to variables before calling
`sql_query()`, and use SQL transformations such as `{sql.val}` for values that
come from SIP messages, headers, URIs, or other external input.