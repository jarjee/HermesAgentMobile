# Proposal: a native chat experience for HermesAgentMobile

> **Status:** investigation / architecture proposal — not finished code.
> **Scope:** add a native Flutter chat screen (and an optional management WebView) to the existing app, plus a dev environment and integration tests. **No runtime change, no rebrand, no ABI changes** — this is an additive feature on top of the current `hermes-agent` stack.

---

## 1. Summary & goals

`HermesAgentMobile` already does the hard part: it bootstraps an Ubuntu 24.04 rootfs via bundled **proot** (no root, no Termux), installs Python 3 + a venv, clones [`nousresearch/hermes-agent`](https://github.com/nousresearch/hermes-agent), and runs its messaging gateway (`gateway/run.py`) as a long-lived foreground service. The agent's HTTP surface is reachable at `127.0.0.1:18789`. But the app surfaces it only as: gateway start/stop, a built-in `xterm` terminal, an onboarding terminal (`hermes setup`), and a log viewer. **There is no in-app chat** — the README tells users to point a browser at `localhost:18789`.

This proposal adds:

1. **A native Flutter `ChatScreen`** that talks to hermes-agent directly over its OpenAI-compatible REST/SSE API on `:18789` — streaming replies, tool-call / "thinking" surfacing, approval prompts, model selection — restyled to the app's existing black/white + `#DC2626` red Material-3 theme. The layout/UX borrows from [`ishandeveloper/Chatter-App`](https://github.com/ishandeveloper/Chatter-App) (a Flutter chat sample) as a *visual skeleton only*.
2. **An optional "Dashboard" WebView tab** wrapping hermes-agent's web management UI (config / env / models / sessions / cron / logs). Lower priority — see the caveat in §4.
3. **A reproducible dev environment + integration tests** that step through the app and assert init/bootstrap works end-to-end.

### Non-goals

- Not pivoting away from `hermes-agent`; not changing the runtime, bootstrap, branding, or supported ABIs.
- Not removing the built-in terminal or the `hermes setup` onboarding terminal.
- Not porting hermes-agent's editor / TUI / cron / skills management natively — that's what the optional Dashboard WebView is for.
- The parallel Termux-CLI path (`bin/hermesx`, `lib/index.js`, `lib/installer.js`, `install.sh`) is untouched — it has no UI surface to add chat to.
- The richer JSON-RPC-over-WebSocket transport (`/api/ws`, see §3) is documented as a future upgrade path, not built in v1.

---

## 2. Background: how the app works today

### 2.1 The Flutter app

```
SplashScreen ──▶ SetupWizardScreen ──▶ OnboardingScreen ──▶ DashboardScreen
                                                                │
              (also reachable: ConfigureScreen, TerminalScreen, LogsScreen, SettingsScreen)
```

- **`flutter_app/lib/services/bootstrap_service.dart`** — Dio-downloads `ubuntu-base-24.04.3-base-<arch>.tar.gz` (URLs in `flutter_app/lib/constants.dart`), extracts it via `NativeBridge.extractRootfs()` (pure-Java tar extraction in `BootstrapManager.kt`), `apt-get install ca-certificates git python3 python3-venv python3-pip curl wget`, `git clone …hermes-agent`, `python3 -m venv venv && pip install -r requirements.txt`, then verifies `/root/hermes-agent/gateway/run.py` exists (`bootstrap_service.dart:195`).
- **`flutter_app/android/.../ProcessManager.kt`** — two proot modes (`buildInstallCommand` ≈ proot-distro's `run_proot_cmd`, `buildGatewayCommand` ≈ `command_login`), bundled `libproot.so` / loaders from `scripts/fetch-proot-binaries.sh`, `env -i` clean guest env, fake `/proc`+`/sys`, resolv.conf bind-mount.
- **`flutter_app/android/.../GatewayService.kt`** — foreground service; launches `cd /root/hermes-agent && source venv/bin/activate && exec python gateway/run.py` (`GatewayService.kt:217`); a port-in-use guard / adopt-existing logic on `18789`; a watchdog + auto-restart (max 5, exponential backoff) that probes "port 18789"; streams stdout/stderr to an `EventChannel` (`com.nxg.hermesagentmobile/gateway_logs`).
- **`flutter_app/lib/services/native_bridge.dart`** — `MethodChannel com.nxg.hermesagentmobile/native`: `getArch/getFilesDir/getNativeLibDir`, `extractRootfs`, `runInProot(cmd,timeout)`, `startGateway/stopGateway/isGatewayRunning`, `setupDirs/writeResolv`, `read/writeRootfsFile`, storage & battery-opt helpers, terminal/setup service controls.
- **UI / theme** — `flutter_app/lib/app.dart` `AppColors`: accent `#DC2626`; dark `#0A0A0A` / `#121212` / `#1A1A1A`, border `#2A2A2A`; light `#FFFFFF` / `#F9F9F9`, border `#E5E5E5`; status green `#22C55E` / amber `#F59E0B` / red `#EF4444` / grey `#6B7280`. Material-3, Google Fonts **Inter** for text, `DejaVuSansMono` for the terminal. Zero-elevation bordered cards (12 px radius), pill status badges, red filled / outlined-secondary buttons. `dashboard_screen.dart` is a `ListView` of "Quick Action" cards + a gateway-status card; `widgets/gateway_controls.dart` has the start/stop card + an "open dashboard" button → `AppConstants.gatewayUrl`; `models/gateway_state.dart` already carries a `dashboardUrl` field.
- **`pubspec.yaml`** already depends on `webview_flutter`, `web_socket_channel`, `http`, `dio`, `provider`, `uuid` — a WebView + HTTP clients are available but currently unused in the Hermes flows. `assets/websscreen.png` exists (OpenClaw lineage).
- **Tests / CI: none.** `.github/` has only `FUNDING.yml` + `dependabot.yml`. A prior `flutter-build.yml` workflow was added then removed because the push token lacked `workflow` scope.

### 2.2 hermes-agent has **three** HTTP surfaces — don't conflate them

| Component | Entrypoint | Framework | Default bind | Role |
|---|---|---|---|---|
| **Messaging gateway** | `gateway/run.py` (`python -m gateway.run`) | core daemon | none — it's a daemon | Bridges the agent to Telegram / Discord / Slack / WhatsApp / Signal / Matrix / email via `gateway/platforms/*.py`. **This is what the app runs.** |
| **`api_server` platform adapter** | `gateway/platforms/api_server.py` | **aiohttp** | core default `127.0.0.1:8642` | **OpenAI-compatible REST + SSE.** Only starts if the `api_server` platform is enabled in `~/.hermes/config.yaml`. **The app assumes this is on `:18789`** (README; `constants.dart:21 gatewayPort = 18789`; `GatewayService.kt`'s port checks / watchdog / notification all hard-code `18789`). |
| **Web dashboard** | `web/` (Vite + React 19 + TS) → `hermes_cli/web_dist/`; backend `hermes_cli/web_server.py` (`python -m hermes_cli.main web` / `hermes dashboard`) | **FastAPI** | `127.0.0.1:9119` | Config / env / model / session / cron / logs / analytics management SPA. **Not run by the app today.** Its "Chat" tab is **not** a chat UI — it's an `@xterm/xterm` terminal over a `/api/pty` WebSocket running the Ink TUI. |

### 2.3 ⚠️ The linchpin question

**The native chat is only possible if something is actually serving HTTP on `:18789`.** `gateway/run.py` itself does *not* serve HTTP — the `api_server` adapter only starts when `platforms.api_server` is enabled in `~/.hermes/config.yaml`, and the repo doesn't obviously do that: `onboarding_screen.dart:103` / `configure_screen.dart:91` just run `python -m hermes_cli.main setup` plain. Nothing in core defaults `api_server` to port `18789` (core default is `8642`; `18789` only appears in `hermes_cli/status.py` as "the gateway port"). So `:18789` may be inherited from the OpenClaw lineage and only work if the user happens to pick that port in the wizard.

**Resolution:** the Phase 0 spike (§7) verifies what's actually listening on `:18789` and what's in `~/.hermes/config.yaml`. If the adapter isn't up by default, the proposal's Part A grows a small bootstrap step: write `platforms.api_server: { enabled: true, host: 127.0.0.1, port: 18789 }` into `~/.hermes/config.yaml` and an `API_SERVER_KEY` into `~/.hermes/.env` (and ensure the launch reads them). Either way the chat feature is achievable; this just determines whether it's "use what's there" or "use what's there + one config-write."

---

## 3. The target API — `gateway/platforms/api_server.py` on `:18789`

Route table (registered in `start()`, `api_server.py:3336-3361`):

| Method & path | Purpose |
|---|---|
| `GET /health`, `/v1/health`, `/health/detailed` | liveness |
| `GET /v1/models` | lists `hermes-agent` as a model (OpenAI shape) — **model picker** + "is a provider configured?" probe |
| `GET /v1/capabilities` | machine-readable capability map (auth required?, header names, endpoint list) |
| `POST /v1/chat/completions` | OpenAI Chat Completions. Body `{model, messages:[{role,content}], stream?}`. `stream:false` → JSON; `stream:true` → SSE (`data: {chat.completion.chunk}\n\n`). Response headers `X-Hermes-Session-Id`, `X-Hermes-Completed`, `X-Hermes-Partial`. Request headers `X-Hermes-Session-Id` (continuity), `X-Hermes-Session-Key` (long-term-memory scoping). |
| `POST /v1/runs` | start an agent run; `202` + `{run_id}` immediately |
| `GET /v1/runs/{run_id}` | poll run status |
| `GET /v1/runs/{run_id}/events` | **SSE stream of structured lifecycle events** — message / message-delta / tool-call / tool-result / "thinking" / approval-request / status / error |
| `POST /v1/runs/{run_id}/approval` | resolve a pending tool / sudo approval |
| `POST /v1/runs/{run_id}/stop` | interrupt a running agent — **the "stop" button** |
| `POST /v1/responses`, `GET/DELETE /v1/responses/{id}` | OpenAI Responses API (stateful via `previous_response_id`) |
| `GET/POST /api/jobs`, `GET/PATCH/DELETE /api/jobs/{id}`, `POST /api/jobs/{id}/{pause,resume,run}` | cron management |

**Transport choice for the chat:**

- **Primary — `POST /v1/runs` → SSE `GET /v1/runs/{run_id}/events`.** hermes-agent is *agentic* (it calls tools), so the structured event stream is what makes a "proper" chat: you can render tool calls / tool results / "thinking" as distinct collapsible elements, and surface approval prompts inline. Send `{messages, model, ...}` to `/v1/runs`, get a `run_id`, open an `EventSource`-style SSE connection to `/v1/runs/{run_id}/events`, fold message-deltas into the in-flight bubble.
- **Fallback — `POST /v1/chat/completions` with `stream:true`.** If the installed hermes version doesn't expose `/v1/runs` + `/events`, fall back to plain OpenAI streaming (token deltas only — no tool/thinking events). The client should feature-detect via `GET /v1/capabilities` / a probe.

**Conversation model:** stateless by default; opt-in server-side continuity via the `X-Hermes-Session-Id` header (sessions persist in SQLite under `~/.hermes/`). **There is no `GET /conversations` / `/messages` history endpoint on the api_server adapter** — so v1 keeps chat history **client-side** (a small local store keyed by session id). (If we later also run `hermes dashboard`, `GET /api/sessions` / `GET /api/sessions/{id}/messages` become available — noted as a future enhancement.)

**Auth / CSRF:** `_check_auth()` (`api_server.py:689`) — `Authorization: Bearer <API_SERVER_KEY>`, constant-time compare. **Wide open if `API_SERVER_KEY` is unset** (it logs a warning; refuses to start on a non-loopback bind without a key). CORS middleware present; **no CSRF**. On loopback-only Android this is acceptable, but the proposal recommends the bootstrap generate an `API_SERVER_KEY` into `~/.hermes/.env` and have the Flutter client send it. The client always uses `127.0.0.1`.

**Onboarding:** `python -m hermes_cli.main setup` (`hermes_cli/setup.py`) is an interactive wizard: model + provider + API key, terminal backend, agent settings, messaging platforms, tools. It persists API keys → `~/.hermes/.env`, everything else → `~/.hermes/config.yaml`, and prints `✓ Setup Complete!` (`setup.py:576`) — already the completion regex `onboarding_screen.dart` / `configure_screen.dart` watch. `--non-interactive` / `--quick` exist but the headless path mostly just *prints guidance*; there's no real provisioning API. So onboarding stays terminal-based; the new chat screen just needs a graceful "no provider configured — run setup" state (detect via `GET /v1/models` / `GET /v1/capabilities`).

**A note on the dashboard's `/api/ws` JSON-RPC (future, out of scope):** `hermes_cli/web_server.py` mounts a newline-delimited JSON-RPC 2.0 WebSocket at `/api/ws` (`tui_gateway/ws.py`), explicitly built "for iOS / web clients" — `session.create` / `prompt.submit` + `message.delta` / `thinking.delta` / `tool.*` events, `session.list/resume/history`, `model.options/save_key`, `slash.exec`. It's richer than the REST surface but requires running `hermes dashboard` on `:9119` and scraping a rotating `__HERMES_SESSION_TOKEN__` from its HTML. The Dart client should be structured so a `HermesWsClient` could later be swapped in behind the same `ChatProvider` — but v1 targets the `:18789` REST/SSE only.

---

## 4. Part A — Native `ChatScreen` (against `:18789` REST/SSE)

### 4.1 New files

| File | Responsibility |
|---|---|
| `flutter_app/lib/services/hermes_client.dart` | REST + SSE client for `:18789`: `listModels()`, `capabilities()`, `startRun({messages, model, sessionId})` → `run_id`, `streamRunEvents(runId)` → typed event stream, `stopRun(runId)`, `resolveApproval(runId, decision)`; plus `chatCompletionsStream(...)` fallback. Sends `Authorization: Bearer …` if configured; threads `X-Hermes-Session-Id`. SSE via a small reader over `http`/`dio` (no WebSocket). |
| `flutter_app/lib/providers/chat_provider.dart` | `ChangeNotifier` (mirrors the existing `GatewayProvider` pattern): message list, in-flight run state, streaming-delta accumulator, session id, model selection; persists the message log locally (no server history endpoint). |
| `flutter_app/lib/models/chat_message.dart` | message model + variants: `user`, `assistant`, `thinking`, `toolCall`, `toolResult`, `approvalRequest`, `status`, `error`. |
| `flutter_app/lib/screens/chat_screen.dart` | `Scaffold` + `AppBar` (title + small gateway-status strip; `Drawer`/overflow with "New chat", "Open Dashboard (web)", model picker) + body `Column[ MessageList(), MessageComposer() ]`. Layout cribbed from Chatter-App's `chatterScreen.dart`; visuals 100% `AppColors` / Inter. |
| `flutter_app/lib/widgets/message_bubble.dart` | user vs. assistant bubbles (asymmetric `BorderRadius` "tail" toggled by sender, à la Chatter-App's `MessageBubble`, recolored to `AppColors`); markdown + monospace code blocks; collapsible "thinking" and "tool call" / "tool result" cards; an inline approval-request card with Approve / Deny → `resolveApproval`. |
| `flutter_app/lib/widgets/message_composer.dart` | multiline `TextField` + circular send button (Chatter-App style); a "stop" button (→ `stopRun`) shown while a run is active. |
| `flutter_app/lib/screens/dashboard_webview_screen.dart` *(optional — Part B)* | themed `Scaffold` wrapping `webview_flutter` `WebViewWidget`. |

### 4.2 Touched files (additive)

- `flutter_app/lib/constants.dart` — add API base-URL helpers (`http://127.0.0.1:18789`) and an optional `apiKey` getter (already has `gatewayPort = 18789`).
- `flutter_app/lib/screens/dashboard_screen.dart` — new "Chat" (and optional "Dashboard (web)") quick-action cards; auto-navigate to `ChatScreen` when `GatewayState` becomes `running`.
- `flutter_app/lib/widgets/gateway_controls.dart`, `flutter_app/lib/models/gateway_state.dart` — wire an "Open Chat" button to the new screen (reuse the existing `dashboardUrl` field).
- `flutter_app/lib/screens/onboarding_screen.dart` — after `✓ Setup Complete!`, offer "Open Chat" beside (or instead of) "Open Dashboard"; flow otherwise unchanged.
- `flutter_app/pubspec.yaml` — add `flutter_markdown` (or hand-roll a minimal markdown/code-block renderer) and `integration_test` (dev dep). `webview_flutter` / `web_socket_channel` / `http` / `dio` / `provider` / `uuid` are already present.
- *(Possibly, Phase-0-dependent)* `flutter_app/lib/services/bootstrap_service.dart` — append a step that writes `platforms.api_server` config into `~/.hermes/config.yaml` and an `API_SERVER_KEY` into `~/.hermes/.env` if the adapter isn't enabled on `:18789` by default.

### 4.3 Behaviour

- **Send:** `POST /v1/runs` with the conversation `messages` + selected `model` + `X-Hermes-Session-Id` → `run_id` → open SSE `GET /v1/runs/{run_id}/events`. Fall back to `POST /v1/chat/completions` `stream:true` if `/v1/runs` is unavailable.
- **Stream:** fold message / message-delta events into the in-flight assistant bubble; render `thinking` events as a collapsible "Thinking…" card; render `tool_call` / `tool_result` events as collapsible cards (tool name + args / result); on `approval_request` (and `sudo` / `secret` requests) show an inline action card → `POST /v1/runs/{run_id}/approval`. On `error`, show an error bubble with a retry affordance.
- **Stop:** while a run is active, the composer shows a stop button → `POST /v1/runs/{run_id}/stop`.
- **History:** persist the message log client-side, keyed by `X-Hermes-Session-Id`; "New chat" starts a fresh session id. (Reconciliation with messages sent via the built-in terminal/TUI is out of scope — the chat screen owns its own session.)
- **Model picker:** `GET /v1/models`; if it returns nothing usable, show a "No provider configured — run Setup" empty state linking to the onboarding terminal.
- **Auth:** send `Authorization: Bearer <API_SERVER_KEY>` if a key is configured (recommended); always `127.0.0.1`.
- **Reuse:** the existing ANSI / box-drawing strippers, the snackbar / clipboard patterns from `terminal_screen.dart`, and the `webview_flutter` / `http` deps already in `pubspec.yaml`.

### 4.4 Effort & reuse

| Bucket | Estimate |
|---|---|
| `HermesClient` (REST + SSE reader + typed events) | ~1–1.5 days |
| `ChatProvider` + `chat_message.dart` + local persistence | ~1 day |
| `ChatScreen` + `MessageBubble` + `MessageComposer` (restyled, markdown, code) | ~2–3 days |
| Tool-call / thinking cards + approval / sudo / secret modals | ~1–2 days |
| Model picker, "no provider" empty state, dashboard card, auto-open, onboarding hook | ~1 day |
| Polish / edge cases (reconnect, long runs, scroll, accessibility) | ~1–2 days |
| **Total — usable v1** | **~3–4 days**; **polished — ~1.5–2 weeks** |

Net-new code only — no existing widget is rewritten. The bootstrap / proot / gateway layer is untouched (or grows one optional config-write step).

---

## 5. Part B — Optional "Dashboard" WebView tab

A themed `Scaffold` wrapping `webview_flutter` `WebViewWidget` pointed at hermes-agent's web dashboard (`http://127.0.0.1:9119/`) — config / env / model / session / cron / logs / analytics management. Reachable from a "Dashboard (web)" card on the home screen and the Chat overflow menu. Handles back-navigation; opens external links via `url_launcher`. The `assets/websscreen.png` art (OpenClaw lineage) is reusable here.

**Caveat — this needs a second process.** The dashboard is `python -m hermes_cli.main web` / `hermes dashboard` on `:9119`, which the app does **not** run today. Part B therefore also requires launching `hermes dashboard` — either by extending `GatewayService.kt`, adding a small second foreground service, or launching it on-demand when the user opens the tab — plus handling the rotating session-token cookie/param the dashboard uses for its own REST. (Pointing a WebView at `:18789` instead is pointless — that adapter serves only JSON/SSE, no HTML.) Because of the extra moving part, this is **optional and phased after the native chat**.

---

## 6. Part C — Dev environment & integration tests

### 6.1 Dev environment

A reproducible setup pinning Flutter, Android SDK / cmdline-tools, JDK 17, an emulator AVD image, and the proot-binary fetch step (`scripts/fetch-proot-binaries.sh`). **Primary: a `Makefile` + `scripts/dev-setup.sh`** (low barrier): `make setup`, `make emulator`, `make run`, `make test`, `make integration-test`, `make apk`. **Optional: a `devenv.nix` / `flake.nix`** for Nix users. Documents `flutter pub get`, `scripts/fetch-proot-binaries.sh`, `scripts/build-apk.sh`.

### 6.2 Integration tests

Add `dev_dependencies: integration_test` (sdk) + an `integration_test/` directory. **Headline test:** launch on an emulator, drive `SplashScreen → SetupWizardScreen`, tap **Begin Setup**, assert the bootstrap state machine progresses (`SetupStep` transitions / `isBootstrapComplete`) and the gateway reaches `running` with `:18789` reachable, then open `ChatScreen` and assert it loads (and, with a stub server, that a streamed reply renders).

A real bootstrap is ~300 MB + several minutes, so the tests are layered:

- **(a) Widget tests** — `SetupState` / `GatewayState` / `ChatProvider` with `NativeBridge` and `HermesClient` mocked. Run on every PR. Fast.
- **(b) A "fast-bootstrap" hook** — an env / build flag that points the rootfs + agent fetch at a pre-baked cached tarball and starts a stub `:18789` server (mock `/v1/models`, `/v1/runs`, `/v1/runs/{id}/events`). So the emulator integration test runs in CI minutes.
- **(c) Optional nightly** — the full real bootstrap via `reactivecircus/android-emulator-runner` (KVM).

### 6.3 CI

`.github/workflows/ci.yml` — `flutter analyze` + `flutter test` on every PR; the emulator integration test (fast-bootstrap mode, KVM runner) on PRs; the full bootstrap nightly; keep `scripts/build-apk.sh` (or promote it to a release workflow). **Note:** re-adding workflow files needs a push token with `workflow` scope (the prior `flutter-build.yml` was removed for exactly this reason).

---

## 7. Risks & open questions

1. **The `:18789` question (Phase-0 gate).** Is `platforms.api_server` enabled on port `18789` after the app's `hermes setup` run, or must the bootstrap write that config? If the latter, Part A grows one bootstrap step (config + `API_SERVER_KEY`). Either way the feature is achievable.
2. **hermes-agent API stability.** v`0.13.0`; pin the cloned commit/tag in the bootstrap so the API doesn't drift under the app.
3. **`/v1/runs` availability.** Does the installed hermes expose `/v1/runs` + `/events`, or only `/v1/chat/completions`? The transport fallback covers both, but verify in Phase 0.
4. **No server-side chat history on the adapter** → client-side persistence; messages sent via the built-in terminal/TUI won't appear in the chat screen (acceptable — different surfaces).
5. **Auth.** Recommend generating `API_SERVER_KEY` even on loopback; minor bootstrap change.
6. **Approval / sudo / secret flows on a small screen** — design the inline cards / modals carefully.
7. **Battery-optimization killing the gateway** — pre-existing failure mode, not introduced here, but the chat screen should surface "gateway not running" clearly.
8. **The optional Dashboard WebView needs a second process** (`hermes dashboard`) — extra moving part; that's why it's optional.
9. **CI cache size** if a pre-baked rootfs is vendored for the fast-bootstrap test.

---

## 8. Phased roadmap

- **Phase 0 — spike (GO / NO-GO).** In the app's proot Ubuntu: start `python gateway/run.py`; `curl -s http://127.0.0.1:18789/health`, `GET /v1/capabilities`, `GET /v1/models`; `cat ~/.hermes/config.yaml` to see whether `platforms.api_server` is enabled on `18789`; `POST /v1/runs` and `curl -N http://127.0.0.1:18789/v1/runs/<id>/events`; `POST /v1/chat/completions` `{"stream":true,...}`. Record the transcript + the real `/v1/runs/{id}/events` event names in an appendix. **If nothing serves `:18789`,** the bootstrap is extended to enable `api_server` on `18789` (still in scope — a slightly bigger Part A).
- **Phase 1 — chat MVP.** `HermesClient` + `ChatProvider` + `ChatScreen`: send via `/v1/runs` (fallback `/v1/chat/completions`), stream events, render user/assistant bubbles, stop button, "Chat" dashboard card, auto-open on `running`.
- **Phase 2 — agentic UX.** Thinking / tool-call / tool-result collapsible cards; approval / clarify / sudo modals; model picker; session-continuity + client-side history persistence; markdown / code rendering; "no provider — run setup" empty state; post-onboarding "Open Chat".
- **Phase 3 — optional Dashboard WebView.** WebView tab + launching `hermes dashboard` on `:9119`.
- **Phase 4 — devenv & CI.** `Makefile` / `scripts/dev-setup.sh` (+ optional `devenv.nix`); widget tests; fast-bootstrap emulator integration test; `.github/workflows/ci.yml`; nightly full-bootstrap job; update `README.md` / `CHANGELOG.md` / `docs/`.

---

## 9. `ishandeveloper/Chatter-App` — what we actually borrow

It's a **Flutter (Dart) chat sample**, not a React/web app: global single-room group chat on **Firebase** (`firebase_auth` + `cloud_firestore`), targeting old Flutter ~1.x / Firebase APIs (`FirebaseUser`, `cloud_firestore: ^0.13.5`, SDK `>=2.1.0 <3.0.0`) — **it won't build on a modern Flutter SDK without migration.** MIT.

The chat UI is essentially one file, `lib/pages/chatterScreen.dart` (~287 lines): `ChatterScreen` = `Scaffold` + `AppBar` with a `LinearProgressIndicator` + a `Drawer` (`UserAccountsDrawerHeader` + Logout) + body `Column[ ChatStream(), inputRow ]`; the input row is `Material(borderRadius:50, elevation:5, child: TextField)` + a circular send `MaterialButton`; `ChatStream` is a `StreamBuilder` over a Firestore snapshot → reversed `ListView` of `MessageBubble`s; `MessageBubble(msgText, msgSender, bool user)` is a `Material` bubble with an asymmetric `BorderRadius` (the "tail" toggled by `user`), coloured blue (mine) vs white (others), aligned left/right.

**We port the *pattern*** — bubble with sender-toggled tail, reversed auto-scroll list, `Material`-pill composer with a circular send button, drawer for session actions — **as fresh, modern Flutter widgets restyled to `AppColors` / Inter.** We do **not** depend on its code (no Firebase; it predates streaming / tool-calls / markdown entirely — those are net-new for an agentic chat).

---

## 10. Appendix — links

- hermes-agent: <https://github.com/nousresearch/hermes-agent>
- Chatter-App: <https://github.com/ishandeveloper/Chatter-App>
- (Phase 0 will append: the `:18789` `curl` transcript + observed `/v1/runs/{id}/events` event taxonomy + the `~/.hermes/config.yaml` `platforms.api_server` state.)

### Decisions locked in with the owner

1. **Hermes-only — no piclaw pivot.** No Bun, no `armhf` drop, no rebrand. (The piclaw option and its Bun-coupling were explored separately and are out of scope here.)
2. **Native chat targets the `:18789` OpenAI-compatible REST/SSE** (`gateway/platforms/api_server.py`) — primary `POST /v1/runs` + SSE `GET /v1/runs/{id}/events`, fallback `POST /v1/chat/completions` `stream:true`. Not the `/api/ws` JSON-RPC (future upgrade path only).
3. **Chat UX reference = Chatter-App's layout**, reimplemented as fresh widgets in this app's theme.
4. **Dashboard WebView tab = optional / phase-later** (it needs a second `hermes dashboard` process).
5. **Termux-CLI path untouched.**
