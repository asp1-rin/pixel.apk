# Pixel — standalone MilkChoco APK (rooted phone, no GameGuardian, no PC)

A single installable APK that delivers the **full `pixel.exe` feature set** on a
**rooted phone** with MilkChoco (`com.gameparadiso.milkchoco`) installed — and
nothing else. No GameGuardian, no PC, no ADB, no terminal.

```
   Install Pixel.apk
        │
        ▼  open the app
   ┌──────────────┐   login (real auth)   ┌──────────────┐   tap Connect (root)
   │  LOGIN screen │ ───────────────────► │ CONNECT screen │ ───────────────────┐
   └──────────────┘                       └──────────────┘                     │
                                                                                ▼
   phone WebView  ◄───── http://127.0.0.1:27345 ◄──── in-process HTTP/SSE server (src/server.js)
        ▲                  full pixel.exe panel        shadows Frida send()/recv()
        └──────────── unmodified pixel.exe agent (agent/agent.ts) injected by frida-inject
```

## How it works

This is the desktop **`pixel.exe`** tool, unchanged, running on the phone:

- `agent/` (the pixel.exe Frida agent) is injected into MilkChoco by a bundled,
  ABI-matched `frida-inject` (root). It does all the memory work.
- `src/server.js` runs a tiny HTTP/SSE server **inside the game process** and
  shadows Frida's `send()`/`recv()`, so the **unmodified desktop renderer**
  (`web-src/`) is served at `http://127.0.0.1:27345` — every cheat, teleport,
  changer, kicker, ESP/aim, resource tool and setting works, automatically.
- The APK (`app/`) is a thin native shell: a branded **login** screen
  (real auth against the existing backend), then a one-button **Connect** that
  injects with root and loads the panel. See `app/.../MainActivity.kt`,
  `Bridge.kt` (native login + connect), `Injector.kt` (root injection).

> ⚠️ **Root is mandatory.** Reading another app's memory is impossible inside
> the Android sandbox without it — the same requirement GameGuardian has. This
> removes the GameGuardian dependency and every manual step, not root.
> Research / education only.

## Use it

1. Get `Pixel.apk` from GitHub Actions (the `apk` artifact) or a release.
2. Install it on the **rooted** phone (MilkChoco already installed).
3. Open **Pixel** → log in with your key → tap **Connect to MilkChoco** →
   grant the root (`su`) prompt. The full panel loads once injection finishes.

## Build

CI does it all — push and the `apk` job builds `Pixel.apk`
(`npm run build` produces `dist/agent.js` + `bin/frida-inject-*`, which the
Gradle `copyPixelAssets` task bundles into the APK; then `gradle assembleDebug`).

Locally (agent only; the APK needs the Android SDK and is best built in CI):

```bash
npm install && npm run build      # -> dist/agent.js + per-ABI bin/frida-inject
```

A `dist/pixel-*.zip` + `launch.sh` manual-injection bundle is also produced as a
fallback for use without the APK.

## Layout

```
agent/                  pixel.exe Frida agent (+ offsets, types) — unmodified
web-src/                pixel.exe desktop renderer — unmodified (full UI)
src/server.js           in-agent HTTP/SSE bridge (shadows send/recv)
build.cjs, scripts/     build pipeline -> dist/agent.js + per-ABI bundles
launch.sh               manual on-device root injector (fallback)
app/                    the installable APK (login + connect shell -> panel)
  app/src/main/java/com/pixel/mobile/
      MainActivity.kt   branded login -> connect shell, then loads the panel
      Bridge.kt         native real-auth POST + connect trigger
      Injector.kt       root su -> push agent + frida-inject -> inject
  app/src/main/res/     adaptive launcher icon (from the pixel logo), strings
legacy/                 the old GameGuardian tool (Lua + MCO-Remote.apk), kept for reference
```

## Honest status

Verified by the build pipeline: agent + renderer bundling, offset injection,
generated-script syntax. **Not** verifiable from a CI/sandbox and needing one
real pass on a rooted device: on-device runtime (Frida `Socket`, `frida-inject`
under root, SELinux ptrace), the in-app `su` injection, the real login round
trip to the backend, and the Gradle APK build itself (standard recipe, runs in
CI — there is no Android SDK in the dev sandbox).

One parity caveat by design: on the desktop, **ESP / aim-circle visuals** are
painted by a separate always-on-top overlay window over the game. The phone
panel is a normal app, so those *on-game visuals* aren't drawn (a future
SYSTEM_ALERT_WINDOW overlay could add them). The underlying logic still runs in
the agent — aimbot, aim-by-circle targeting, blackhole, teleport, speed,
kicker, changer, resource tools, etc. all work; only the ESP box/tracer drawing
over the live game is absent.
