# Aiko TV

Making a Smart TV a first-class actor on the Aiko Services bus.

> **Status:** design phase. No implementation yet. This repo holds the platform
> analysis and the build sequence; the code lands in the repos it belongs to
> (see [Where the code lives](#where-the-code-lives)).

## Why this repo exists

The work spans three repos and belongs to none of them:

- [`aiko_services_dart`](https://github.com/nickmeinhold/aiko_services_dart) — the
  Dart port that supplies the codec, dispatch, and transport
- `downstream-tizen` — the Samsung TV app that would host the actor
- `downstream-webos` — the LG TV app, which needs a different answer entirely

So the design, the platform findings, and the cross-repo build order live here,
and each repo carries its own slice of the implementation.

## The finding: why Dart and not Python

The instinct is to ask "can the TV run Python?" — but that's the wrong question.
The TV doesn't run Dart either. It runs **machine code** (Tizen) or
**JavaScript** (webOS). Dart's advantage is that it has a downhill compile path
to both, and someone already built the bridge.

Full write-up: [**docs/platform-analysis.html**](docs/platform-analysis.html).

The short version:

| | Tizen (Samsung) | webOS (LG) |
|---|---|---|
| What the platform executes | native ARM machine code | JavaScript in a Chromium webview |
| How Dart gets there | AOT-compiled by `flutter-tizen`, hosted by Samsung's Flutter embedder | `dart2js`, or skip Dart and write JS |
| Python's path | none — no embedder exists | Pyodide (CPython → WASM) would technically load |
| Verdict | **drop `aiko_services_dart` in nearly unchanged** | needs a separate answer |

The first-order obstacle for Python is not the language, it's the **embedder**.
Flutter isn't one of Tizen's supported app models either — `flutter-tizen` exists
because Samsung wrote a native C++ Tizen app that hosts the Flutter engine and
wires up the Evas surface, input, and lifecycle. Nobody has written the
equivalent for CPython. Underneath that sit the second-order problems: CPython
needs an interpreter, a stdlib, and runtime imports (no AOT analogue), and
`numpy` / `pyzmq` / `Pillow` are compiled `.so` files that would each need
cross-compiling against Tizen's ARM sysroot.

webOS is a separate matter: `downstream-webos` was a Flutter Web app until
2026-06-20, when Flutter Web was proven on-device to crash the LG C2's webview
during **engine bootstrap** — a hello-world crashes identically to the full app.
That app is now vanilla HTML/JS with no build step. So the webOS route is either
`dart2js` to a JS bundle (reintroducing a build step) or `mqtt.js` over
WebSockets plus a JS port of the S-expression codec held to the same RFC-0001
vectors.

## The open design question

Everything above is transport. What forks the actual build is **what the TV does
as an actor**:

- **Sink** — the TV subscribes and reacts (`aiko_chat` says "play this", the TV
  plays it). Small: subscribe + dispatch into the existing player.
- **Peer** — the TV publishes state and is discoverable via the registrar.
  Needs the Actor / registrar / Eventual Consistency layer, which isn't ported
  yet (it's items 1–2 of `aiko_services_dart`'s own roadmap).

The transport work is shared by both. Everything above it depends on the answer.

## Build sequence

1. **Transport split** (`aiko_services_dart`) — `lib/src/transport/mqtt_transport.dart`
   imports `mqtt_server_client.dart`, which pulls `dart:io` and makes the package
   non-web-compilable. Conditional-import split now, at the seam, rather than
   discovering it later.
2. **Decide sink vs peer** (here) — the fork above.
3. **Tizen spike** — `aiko_services_dart` as a path dependency in the Tizen app,
   connect to the broker, round-trip one message on real hardware.
4. **Wire into the app** (`downstream`) — the actor drives the existing player.

## Where the code lives

| Slice | Repo |
|---|---|
| Codec, dispatch, MQTT transport, Actor layer | `nickmeinhold/aiko_services_dart` |
| TV app hosting the actor | `downstream` (`downstream-tizen/`) |
| LG TV app | `downstream` (`downstream-webos/`) |
| Design, platform findings, cross-repo order | this repo |

## Upstream

- [geekscape/aiko_services](https://github.com/geekscape/aiko_services) — Python reference implementation
- [geekscape/aiko_chat](https://github.com/geekscape/aiko_chat) — Aiko Services based chat server
- [nickmeinhold/aiko_services_dart](https://github.com/nickmeinhold/aiko_services_dart) — the Dart port ([docs hub](https://nickmeinhold.github.io/aiko_services_dart/))
