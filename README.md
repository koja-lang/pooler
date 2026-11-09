# Pooler

[![CI](https://github.com/koja-lang/pooler/actions/workflows/ci.yml/badge.svg)](https://github.com/koja-lang/pooler/actions/workflows/ci.yml)
[![GitHub Release](https://img.shields.io/github/v/release/koja-lang/pooler)](https://github.com/koja-lang/pooler/releases)

A generic resource pool for [Koja](https://github.com/koja-lang/koja).
Pools any value (database connections, sockets, sessions) behind a
single process that lends resources to callers one at a time.

## Features

- Fixed-size pool built in the background from a factory closure
- Checkout callers queue in FIFO order and block, with a timeout, until a resource frees up
- Checkout hands out a `Lease`. Checkin and discard take it back, so a lease the pool never issued, or one returned twice, is ignored and cannot grow the pool
- Checked-in values replace the lent copy, so in-place updates survive the round trip
- Optional `valid` check at checkin drops broken resources before the next borrower sees them
- Optional `destroy` hook closes every resource the pool lets go of, on an invalid checkin and on stop
- Empty slots refill with the factory. When the factory fails, the pool logs the error and retries on a randomized exponential backoff until it succeeds, the way Elixir's DBConnection does

## Installation

Add the package to your `koja.toml`.

```toml
[dependencies]
pooler = { github = "koja-lang/pooler", tag = "v0.1.0" }
```

## Usage

`Connection` stands in for any resource type. Its `ping` returns the
updated connection on success, the usual shape for a value type.

```koja
alias Pooler.Config
alias Pooler.Error
alias Pooler.Pool

fn start_pool -> Pool<Connection> ! Error
  config = Config{
    create: fn () -> Result<Connection, String> Connection.open() end,
    destroy: Option.Some(fn (conn: Connection) -> () _ = conn.close() end),
    size: 5,
    valid: Option.Some(fn (conn: Connection) -> Bool conn.idle?() end),
  }

  # Fails only when the config is invalid. The pool builds its
  # connections in the background after this returns.
  try Pool.start(config)
end

fn ping(pool: Pool<Connection>) -> Bool ! Error
  # Borrow a connection, waiting up to 5 seconds. Fails with
  # `Error.Timeout` when none frees up in time.
  lease = try pool.checkout(5000)

  match lease.resource.ping()
    Result.Ok(conn) ->
      # Store the updated connection on the lease before handing it
      # back. The next borrower receives what is on the lease.
      lease.resource = conn
      pool.checkin(lease)
      true

    Result.Err(_) ->
      # Hand it back broken. The pool destroys it and builds a
      # replacement.
      try pool.discard(lease, 5000)
      false
  end
end
```

`pool.stop()` destroys the shelved connections and exits. Lent
connections stay with their borrowers.

Every failure is a `Pooler.Error`.

| Variant                 | Meaning                                                                           |
| ----------------------- | --------------------------------------------------------------------------------- |
| `InvalidConfig(String)` | `start` rejected a `Config` field, named in the string                            |
| `Timeout`               | No resource was ready within the checkout timeout, all lent out or none built yet |
| `PoolDown`              | The pool process was stopped                                                      |

A factory that fails at startup does not bring the pool down. Checkouts time out until the first resource is built, and each failed attempt logs one `error` line with the factory's reason and the delay before the next attempt.

## Not yet supported

- Reclaiming a lease from a crashed borrower. The pool does not monitor borrowers, so that lease is never returned and its slot stays empty
- Dynamic resizing and overflow resources
- Health checks on idle resources. `valid` runs at checkin only, never on the shelf

## Development

```sh
koja test
```

## License

Copyright (c) 2026 Henry Popp

This project is MIT licensed.
