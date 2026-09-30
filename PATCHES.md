# PATCHES — rama `quic-zig-0.17`

Esta rama es la dependencia libxev de `MKS2508/quic-zig` (rama `main` tras el rebase
sobre upstream `endel/quic-zig`). Su `build.zig.zon` pinnea el commit **`f5d3811`**
por hash de paquete (`libxev-0.0.0-86vtc161FACqYeIlWiSoja8g74RJxfzAu0yFD1uaUDsr`).
No reescribir la historia hasta ese commit: el pin sólo necesita que el commit exista
en el remoto.

## Base

- `c1e223b` — port de libxev a Zig `0.17.0-dev.1893+78e3b1c73` (rama `feat/zig-0.17`,
  ya contenida en `main`). `main` avanzó después sólo con 3 commits de docs
  (`be583e3`, `bdf97fc`, `0c0754e`), por eso esta rama no es fast-forward de `main`.

## Parches encima de la base (cherry-picks de PRs abiertas en `mitchellh/libxev`)

| PR upstream | Rama del autor | Commits aquí | Qué |
|---|---|---|---|
| [#224](https://github.com/mitchellh/libxev/pull/224) | `pr-224` (Bryan) + `kqueue-fixes` (endel) | `8fab494`, `f266a50`, `3d256f4` | kqueue: `tick(0)` vacía los `EV_DELETE` pendientes; `.rearm` de completions no-kqueue pasa a `.adding`; `next` limpio en `submit`. Tests de reproducción + `zig fmt`. |
| [#245](https://github.com/mitchellh/libxev/pull/245) | `kqueue-rearm-submit` (endel) | `ffeed90` | kqueue: un completion re-armado desde `submit()` sigue activo (antes quedaba `.dead` y `cancel` no hacía nada). |
| [#246](https://github.com/mitchellh/libxev/pull/246) | `accept-nonblock` (endel) | `52f0704` | epoll: `accept` devuelve sockets no bloqueantes (`accept4` + fallback `fcntl`). |
| [#247](https://github.com/mitchellh/libxev/pull/247) | `send-nosignal` (endel) | `8d6003c`, `b166fec` | epoll/io_uring/kqueue: `send` con `MSG_NOSIGNAL`; kqueue mapea `EPIPE`/`ECONNRESET` a `BrokenPipe`/`ConnectionResetByPeer`. |
| [#240](https://github.com/mitchellh/libxev/pull/240) | `iocp-poll` (endel) | `2eb894f` | IOCP: `poll` vía `IOCTL_AFD_POLL`. |
| — (port 0.17) | — | `f5d3811` | Port a 0.17 de los tests de #224 (`@splat`) + `dynamic.zig` a `Union.FieldAttributes`. |

## Estado de verificación

- Suite de tests en Linux (epoll / io_uring) con Zig 0.17.0-dev.1893.
- **Los parches kqueue (#224, #245 y la parte kqueue de #247) NO se han ejecutado en un
  Mac/BSD**: sólo cross-compilan (`aarch64-macos`, `x86_64-macos`). Tampoco se ha
  ejecutado el backend IOCP (#240) en Windows.
- Si alguna PR se mergea upstream, al rebasar sobre `mitchellh/main` hay que soltar el
  cherry-pick correspondiente y mover el pin de `quic-zig`.
