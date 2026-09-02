# libxev — fork notes (MKS2508)

This is **`MKS2508/libxev`**, a fork of `mitchellh/libxev`. It exists to serve
**`mks2508-quic-zig`**, which consumes it as a **pinned tarball**.

## 📍 Planned work lands here, but the plan lives in the `mesh` repo

**`/Volumes/KODAK1TB/REPOS y PROYECTOS/nodejs-bun/mesh`** (`MKS2508/mesh`, branch `main`)

| Read this first | What it is |
|---|---|
| `docs/handoffs/HANDOFF-lanes-2026-09-02.md` | **Start here.** Overview of all 7 planned lanes. |
| `docs/task-requests/quicz-windows.plan.md` | The plan. **M6a and M6b land in THIS repo**; every other milestone is in `quic-zig`. |
| `roadmap.spec.yml` → **D-064** | The decision that put this work here. The ledger is authoritative over prose. |

## D-064 — why work lands in this fork at all

`quic-zig` does not compile for Windows. The structural blocker is here: its event loop
calls `xev.File.poll(.read)` (`event_loop.zig:498,501`), and **this backend does not
implement `poll`** — `src/watcher/stream.zig:101,282,301` are
`@compileError("poll not supported on this backend")`. IOCP is completion-based; the
readiness model that event loop is built on does not exist on Windows.

waxin was offered a cheaper workaround (a `Poller` seam inside `quic-zig`, leaving this
fork untouched) and **chose to fix it here instead**, via the missing `poll` op implemented
over **AFD** — the same NT surface libuv, tokio and mio use for exactly this. The correct
layer, deliberately, over the cheaper one.

### Scope, verified

- **5 sites in `src/watcher/stream.zig`, not 3**: the three `@compileError`s plus
  `PollError` (`:60`, currently `error{}`) and `PollEvent.read` (`:74`, currently
  `0 // invalid`).
- **This fork does not use Zig's `std.os.windows`.** It has its own wrapper
  (`src/windows.zig`, 815 lines) with **zero AFD**. Hence M6a = declarations, M6b =
  integration in `iocp.zig`.
- **Dispatch is free**: `Loop.tick` recovers the Completion via `@fieldParentPtr`
  (`iocp.zig:344-348`) without inspecting the handle, so an AFD completion routes by
  itself. And `Operation`/`Result` are exhaustive unions, so adding `.poll` makes the
  compiler enumerate every remaining switch.
- **Mirror the upstream `poll` shape exactly**: `epoll.zig:1186` and `io_uring.zig:903`
  both declare `poll: PollError!void` and return the events **requested**, not the ones
  received. Doing `@enumFromInt` on the kernel's raw mask is UB — `PollEvent` is an
  exhaustive single-member enum.

### 🔴 Two traps, both found by review rather than by the compiler

1. **`NtCancelIoFileEx` cancels by the `IO_STATUS_BLOCK` pointer used at submit.** A
   separate `iosb` field plus cancelling with `&completion.overlapped` is a **silent
   no-op**. Do what libuv does: the `OVERLAPPED` *is* the `IO_STATUS_BLOCK` (their first
   two fields overlap exactly).
2. **The requested mask cannot be `AFD_POLL_RECEIVE` alone.** A socket that dies without
   its failure bits requested **never completes the poll**, and the loop hangs. Include the
   failure bits.

### R11 — the one thing that cannot be audited offline

`AFD_POLL_INFO` and the `AFD_POLL_*` bitmask are **in no header Zig ships** (grepped).
They are undocumented NT; the only provenance is reading libuv/mio/wepoll. Every other
constant in the plan comes from a citable header. **Mark their provenance in a comment —
never dress them up as header constants** — assert the layout at comptime, and cross-check
against two independent sources. A wrong value only shows up when it runs on real Windows.

## Iterating without re-pinning

`quic-zig` pins this fork by commit + hash in `build.zig.zon:5-8`. During development use
`zig build --fork=<path to this checkout>` (Zig 0.17-dev 1893), which resolves the
dependency against a local checkout — verified to produce an identical error set to the
pinned tarball.

🔴 **Order matters**: commit and **push here first**, `zig fetch --save` in `quic-zig`
**after**. The other way round the sha does not exist yet and the `.zon` is left broken.

## Relationship to the pin, as of 2026-09-02

This fork is **7 commits ahead** of the pinned commit `cedcbf7`, but **`src/` is
byte-identical** — those 7 touch only build and CI. There is no drift to reconcile.
