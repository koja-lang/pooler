# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.1.0] - 2026-11-09

Requires Koja 0.20.

### Added

- Start a fixed-size resource pool with `Pool.start`, backed by a user-supplied factory closure. The pool builds its resources in the background, so `start` fails only on an invalid `Config`, with `Error.InvalidConfig`.
- Borrow resources with `Pool.checkout`, which returns a `Lease`, queueing callers in FIFO order when every resource is lent out.
- Return leases with `Pool.checkin`, preserving updates made to the resource while borrowed. A lease the pool never issued, or one returned twice, is ignored with a warning.
- Hand back broken resources with `Pool.discard`, which destroys them and rebuilds the slot with the factory.
- `Config.valid` runs on every checked-in resource. A value that fails the check is destroyed and its slot refilled.
- `Config.destroy` runs on every resource the pool lets go of, so a connection pool can close its sockets.
- `Pool.stop` destroys the shelved resources before the pool exits.
- When the factory fails, at startup or on refill, the pool logs the error and retries on a randomized exponential backoff bounded by `Config.backoff_min_ms` and `Config.backoff_max_ms`. The pool never shrinks and never gives up.
