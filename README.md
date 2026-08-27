# Kamailio Style Guide for CaptionCall

This guide describes a practical coding style for Kamailio configuration files.
The rules are generalizations and best practices; use common sense above all
else.

The goal is to make Kamailio scripts readable, scannable, and easy to maintain
when they are revisited during incidents, reviews, and feature work.

## General Principles

- Optimize for the person debugging the file at 2 a.m.
- Keep related logic close together.
- Prefer explicit names over clever shortcuts.
- Make control flow easy to scan from top to bottom.
- Avoid formatting churn in unrelated lines when making a small change.

## File Layout

A Kamailio configuration file should be organized in predictable sections:

1. Shebang and preprocessor options.
2. Global parameters.
3. Module loading.
4. Module parameters.
5. Flags, constants, and reusable definitions.
6. Main request route.
7. Named routes.
8. Branch, failure, reply, onreply, and event routes.

Keep the order stable unless a specific dependency requires otherwise. When a
setting depends on another setting, put the dependency first.

### Section Headings

Use short, visible comment headings for major sections.

```kamailio
####### Global Parameters ########

####### Modules ########

####### Routing Logic ########
```

Do not add decorative headings for every small block. Too many headings make the
file harder to scan.

## Naming Conventions

Names should make it clear what kind of value is being used and why it exists.

### Preprocessor Values

For things named with `define`, `subst`, `defenv`, and related preprocessor
keywords:

- Use all upper-case names.
- Separate multiple words with underscores.
- Prefer the `!!` sigil over `#!` after the first line, so preprocessor lines
  are not visually confused with comments or the `#!KAMAILIO` shebang.
- Keep names descriptive enough to survive being read far from the definition.

Good:

```kamailio
#!KAMAILIO

!!define ENVIRONMENT dev
!!define DEPLOY_ENVIRONMENT dev
!!define DISPATCHER_LIST sip_dispatcher
```

Avoid:

```kamailio
#!define env dev
#!define dl sip_dispatcher
```

### Flags

Kamailio flags are integer values. Give them preprocessor constant names so the
meaning is visible at the call site.

Good:

```kamailio
## Near the head of the script.
!!define FLT_ACC 1
!!define FLT_NATS 2

## Later in the config.
setflag(FLT_ACC);
```

Avoid:

```kamailio
## It is difficult to remember what flag 1 means.
setflag(1);
```

Use consistent prefixes when possible:

- `FLT_` for message flags.
- `FLB_` for branch flags.
- `AVP_` for named AVPs when represented by constants.
- `HDR_` for reusable header names.

### Routes

Route names should be upper case and describe the action or decision the route
performs.

Good:

```kamailio
route[RELAY] {
    if ( !t_relay() ) {
        sl_reply_error();
    }
}

route[HANDLE_NAT] {
    force_rport();
}
```

Avoid names that only describe where the route is called from, such as
`route[HELPER]` or `route[ROUTE_1]`.

### Variables

Prefer names that describe the value being stored, not the Kamailio pseudo-
variable mechanism used to store it.

Good:

```kamailio
$var(source_ip) = $si;
$avp(account_id) = $hdr(X-Account-ID);
```

Avoid:

```kamailio
$var(x) = $si;
$avp(tmp) = $hdr(X-Account-ID);
```

### `$var()` vs `$avp()`

Use `$var()` for local scratch values that are set and consumed during the same
synchronous processing path. Treat `$var()` values as process-local working
memory, not as per-message or per-transaction storage. Always assign a `$var()`
before reading it in the same route path, and do not rely on it being `$null` at
the start of request processing.

Use `$avp()` when the value belongs to the SIP message or transaction: values
copied from headers, URI parts, authentication state, database lookups, or any
state that must still be correct in branch, failure, reply, or on-reply routes.
AVPs are usually the safer default for call-specific state.

Good:

```kamailio
$avp(account_id) = $hdr(X-Account-ID);

$var(route_query) = "SELECT destination_uri FROM account_routes "
                  + "WHERE account_id="
                  + $(avp(account_id){sql.val});

if ( sql_query("main_db", $var(route_query), "ra") < 0 ) {
    xerr("route=LOOKUP sql failure callid=$ci account=$avp(account_id)\n");
    sl_send_reply("500", "Lookup Failed");
    exit;
}
```

Avoid:

```kamailio
## This value may be stale or from another message when read later.
$var(account_id) = $hdr(X-Account-ID);
t_on_reply("ACCOUNT_REPLY");

onreply_route[ACCOUNT_REPLY] {
    xinfo("account=$var(account_id) callid=$ci\n");
}
```

## Indentation and Whitespace

- Use four spaces for indentation.
- Do not use tabs.
- Place one blank line between logical blocks.
- Place two blank lines between route blocks.
- Do not leave blank lines immediately after route declarations.
- Keep braces on the same line as route declarations and control statements.
- Use inner white space for condition parentheses: write
    `if ( $var(i) == 0 )`, not `if ($var(i) == 0)`.
- Do not use inner white space for function-call parentheses: write
    `xinfo("message\n")`, not `xinfo( "message\n" )`. This makes
    parentheses used for conditional evaluation easier to distinguish from
    parentheses used for function arguments.

Good:

```kamailio
route[CHECK_SOURCE] {
    if ( $si == "10.0.0.1" ) {
        xinfo("trusted source ip=$si\n");
        return;
    }

    xwarn("untrusted source ip=$si\n");
}


route[HANDLE_SOURCE] {
    route(CHECK_SOURCE);
}
```

Avoid:

```kamailio
route[CHECK_SOURCE]
{

    if ( $si == "10.0.0.1" )
    {
        xinfo("trusted source ip=$si\n");
        return;
    }
}
```

## Line Length

Try to keep lines to 80 characters maximum. If a trailing semicolon or closing
quote would push a line slightly past 80 characters, use judgment. Some
Kamailio expressions do not break cleanly, but make a reasonable effort.

Break long strings by using adjacent string literals when the command supports
it.

Good:

```kamailio
xinfo("This is a long line that exceeds 80 characters in length, and thus "
      "should be broken to multiple lines\n");
```

Debatable, but acceptable when the semicolon is just over the limit:

```kamailio
xinfo("This line places the semicolon at the 81st character. It is debatable.\n");
```

### Breaking Function Calls

When a function call has many arguments, break after commas and align
continuation lines under the first argument. This rule is for function-call
arguments, not conditional expressions.

```kamailio
uac_replace_from("display name",
                 "sip:user@example.com");
```

If the function call is still hard to read, assign intermediate values before
the call.

```kamailio
$var(from_display) = "display name";
$var(from_uri) = "sip:user@example.com";

uac_replace_from($var(from_display), $var(from_uri));
```

### Breaking Boolean Expressions

For long conditions, put each major condition on its own line. Put the boolean
operator at the beginning of each continuation line and align the operators
with each other. This rule is for conditional expressions, not function-call
arguments. Put the closing parenthesis and opening brace on their own line,
aligned with the `if`, so the end of the condition is easy to spot.

```kamailio
if ( $rm == "INVITE"
        && is_method("INVITE")
        && $si != $null
) {
    route(HANDLE_INVITE);
}
```

Prefer extracting repeated or subtle checks into named routes rather than
building one very large condition.

## Comments

Comments should explain why the script is doing something, not restate what the
next line already says.

Good:

```kamailio
## Some endpoints send private Contact addresses even when the Via is correct.
## Fix the Contact before relay so replies do not disappear behind NAT.
fix_nated_contact();
```

Avoid:

```kamailio
## Fix the nated contact.
fix_nated_contact();
```

Use `##` for normal comments. Reserve larger banner comments for major section
headings.

Example larger banner comment:

```kamailio
####### Routing Logic ########
```

Use `/* ... */` sparingly for short block comments when a multi-line note reads
better as one unit.

Good:

```kamailio
/* Preserve the original Request-URI before dispatcher rewrites it so failure
 * handling can log the caller's requested destination.
*/
$avp(original_ruri) = $ru;
```

Avoid using block comments for routine one-line notes.

```kamailio
/* Fix the nated contact. */
fix_nated_contact();
```

## Logging

Logs should be useful during production troubleshooting. Include the identifying
fields that let a reader connect the log line to a SIP message or transaction.

- Use `xinfo()` for normal lifecycle events.
- Use `xwarn()` for unusual but recoverable conditions.
- Use `xerr()` for failures that require attention.
- Prefer the shorter level-specific forms, such as `xinfo("message\n")`, over
    `xlog("L_INFO", "message\n")`.
- Include values such as `$ci`, `$rm`, `$ru`, `$si`, `$sp`, and route context
  when they help identify the message.
- End log messages with `\n`.

Good:

```kamailio
xinfo("route=INVITE callid=$ci method=$rm ruri=$ru source=$si:$sp\n");
```

Avoid:

```kamailio
xinfo("got here\n");
xlog("L_INFO", "route=INVITE callid=$ci method=$rm ruri=$ru source=$si:$sp\n");
```

Avoid logging secrets, credentials, tokens, or full payloads unless there is a
specific debugging need and the log destination is appropriate.

## Control Flow

Keep route control flow direct and explicit.

- Return early for guard clauses.
- Prefer named routes for repeated logic.
- Keep the main `request_route` high level.
- Avoid deeply nested `if` blocks.
- Make every `drop`, `exit`, `return`, and `sl_send_reply()` easy to justify
  from nearby context.

Good:

```kamailio
request_route {
    route(SANITY_CHECKS);

    if ( is_method("OPTIONS") && $rU == $null ) {
        sl_send_reply("200", "Keepalive");
        exit;
    }

    route(WITHINDLG);
    route(AUTH);
    route(RELAY);
}
```

Avoid packing unrelated behavior into the main request route. A reader should be
able to skim the main route and understand the broad request lifecycle.

## Route Structure

Named routes should do one job. If a route validates input, mutates headers,
selects a destination, and relays the request, it is probably too large.

Within a route, order the logic as:

1. Preconditions and guard clauses.
2. Local variable setup.
3. Main behavior.
4. Exit or return behavior.

Example:

```kamailio
route[HANDLE_REGISTER] {
    if ( !is_method("REGISTER") ) {
        return;
    }

    $var(source) = $si + ":" + $sp;
    xinfo("route=REGISTER callid=$ci source=$var(source)\n");

    save("location");
    exit;
}
```

## Module Parameters

Keep module parameters grouped by module and close to the corresponding
`loadmodule` section when practical.

Good:

```kamailio
loadmodule "tm.so"
loadmodule "sl.so"

modparam("tm", "failure_reply_mode", 3)
modparam("tm", "fr_timer", 30000)
```

When a parameter value is environment-specific, prefer a named preprocessor
constant over repeating the literal value throughout the file.

```kamailio
!!define FR_TIMER_MS 30000

modparam("tm", "fr_timer", FR_TIMER_MS)
```

## SIP Headers

- Use constants for repeated custom header names.
- Normalize header handling in one route when several routes need the same
  behavior.
- Be careful when removing or rewriting headers; include a comment explaining
  interoperability or security reasons.

Good:

```kamailio
!!define HDR_ACCOUNT_ID X-Account-ID

$avp(account_id) = $hdr(HDR_ACCOUNT_ID);
```

## Error Handling

Prefer explicit failure handling over falling through accidentally.

Good:

```kamailio
if ( !ds_select_dst(DISPATCHER_SET, "4") ) {
    xwarn("route=DISPATCHER no destination callid=$ci ruri=$ru\n");
    sl_send_reply("503", "No Destination Available");
    exit;
}
```

When relaying, handle relay failure in a consistent helper route.

```kamailio
route[RELAY] {
    if ( !t_relay() ) {
        sl_reply_error();
    }

    exit;
}
```

## SQL

When using SQL through `sqlops`:

- Use stable, descriptive connection names so each `sql_query()` call is easy
    to match to its configured connection.
- Log enough context to identify the SIP transaction and SQL operation.
- Handle lookup misses separately from hard failures when the module allows it.
- Assign query strings to variables before calling `sql_query()` so the
    operation is named and the function call stays readable.
- Use SQL transformations such as `{sql.val}` for any value that came from a
    SIP message, header, URI, or other external source. Do not concatenate raw
    pseudo-variables into quoted SQL strings.
- Keep query construction readable; avoid hiding important behavior in a long
  one-line string.

Example:

```kamailio
modparam("sqlops", "sqlcon", "main_db=>mysql://user:pass@db/main")
modparam("sqlops", "sqlcon", "cdr_db=>mysql://user:pass@db/cdr")

$var(route_query) = "SELECT destination_uri FROM account_routes "
                  + "WHERE account_id="
                  + $(avp(account_id){sql.val});

if ( sql_query("main_db", $var(route_query), "ra") < 0 ) {
    xerr("route=LOOKUP sql failure callid=$ci account=$avp(account_id)\n");
    sl_send_reply("500", "Lookup Failed");
    exit;
}
```

## Tests and Validation

Before committing a Kamailio config change, run the cheapest validation that
can catch syntax or preprocessing mistakes.

Recommended checks:

```sh
kamailio -f path/to/kamailio.cfg -c
```

If the config depends on environment variables, definitions, or mounted files,
run the check in the same container or deployment environment that supplies
those dependencies.

For formatting-only changes, review the diff and verify that no route behavior
changed.

## Review Checklist

Use this checklist before sending a change for review:

- Are constants named clearly and consistently?
- Are magic flag numbers avoided?
- Is the main route easy to skim?
- Are long lines wrapped where practical?
- Are log lines useful and free of secrets?
- Are exits and replies intentional?
- Did you run `kamailio -c` or the closest available validation?
- Is the diff limited to the intended behavior or formatting change?
