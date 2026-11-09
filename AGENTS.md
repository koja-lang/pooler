# Agent Guide

This is a [Koja](https://kojalang.org) project. Koja is a statically
typed, compiled language with Ruby-inspired syntax, value semantics,
and Erlang-style concurrency.

## This package

`Pooler` is a generic resource pool. One process holds `size` values
built by a factory closure and lends them to callers one at a time.
The public surface is `Config`, `Lease`, `Error`, and `Pool`, one per
file in `src/`. `src/server.koja` holds the pool process and its
message types. `Msg`, `Reply`, and `Lease.Token` are public only
because public fields name them, and `@doc false` keeps them out of
the docs. Nothing outside this package should construct them.

Startup and refill follow Elixir's DBConnection. The pool builds in
the background, a failing factory logs one error per attempt and
retries forever on a randomized exponential backoff, and `PoolDown`
means only that the pool was stopped.

Tests mirror `src/`. `test/pool_test.koja` covers the contract of each
public method, `test/lease_test.koja` covers lease identity, and
`test/server_test.koja` covers refill and backoff. Shared helpers live
in `test/support/`. `koja test` compiles every file under `test/`, so
a `priv` declaration there is visible to every test file.

`Log` is experimental in Koja 0.20, so `koja.toml` sets
`experimental = true`. `CHANGELOG.md` follows Keep a Changelog.

## Look up documentation

`koja doc <symbol>` prints the full doc for any stdlib or project
symbol, like `koja doc List.append`. `koja doc search <query>`
lists every symbol whose name or doc mentions the query.

The standard library sources are extracted to `~/.koja/stdlib/`,
one directory per compiler build. Read or grep them there when the
docs are not enough.

## Commands

Set `KOJA_DIAGNOSTICS=short` to print each diagnostic on one line.

- `koja check` type checks without compiling
- `koja test` runs `test "description"` blocks from `src/` and
  `test/`. `assert expr` fails the test when `expr` is false, and
  `fail "message"` fails it outright. Reaching `end` passes.
- `koja run` builds and executes the project
- `koja format` formats the project in place (`--check` to verify)

## Language essentials

- No `let`, `var`, `mut`, or semicolons. Assignment creates a
  variable, and blocks close with `end`.
- No `else if`. Use `cond` for multi-branch conditionals.
- Values never alias. A mutating function takes `self` and returns a
  new value, so rebind the result, as in `list = list.append(42)`.
- `-> T ! E` declares a fallible function. `try expr` unwraps or
  propagates the error, `fail e` returns an error, and
  `expr rescue e -> handler` handles it inline.
- Files in one package share a namespace. Same-package code needs no
  imports. Use `alias Pkg.Type` to shorten external package paths.
