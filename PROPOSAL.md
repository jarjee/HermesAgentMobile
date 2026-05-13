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

### 2.3 ⚠️ The linchpin — verified empirically (Phase 0 done, 2026-05-13)

**The native chat is only possible if something is actually serving HTTP on `:18789`.** The Phase-0 spike was run; results are unambiguous:

- **As shipped, `python gateway/run.py` opens *zero* ports.** With a fresh `~/.hermes` and no env vars, the gateway boots, logs `"No messaging platforms enabled"`, and idles. `ss -tln` shows nothing on `:18789`, `:8642`, or `:9119`; every `curl` to those ports gets `connection refused`. **The app's current `localhost:18789` premise is broken in practice** — the watchdog at `GatewayService.kt:381` ("port 18789 not responding") would always trigger, the "Browser at localhost:18789" line in `README.md:29` is fiction. The user has confirmed this matches their hands-on experience.
- **The fix is a 4-line env-var injection.** `gateway/config.py:1402-1426` enables the `api_server` platform iff `API_SERVER_ENABLED` (truthy) **or** `API_SERVER_KEY` is set; port is read from `API_SERVER_PORT` (core default `8642`, not `18789`). Setting:
  ```
  API_SERVER_ENABLED=true
  API_SERVER_PORT=18789
  API_SERVER_KEY=<generated>
  GATEWAY_ALLOW_ALL_USERS=true       # otherwise the agent denies its own runs
  ```
  in `~/.hermes/.env` (which `python-dotenv` auto-loads — verified) is sufficient to bring up the full chat API. No `config.yaml` schema edits required.
- **`hermes setup` does not wire api_server.** `grep -n "api_server" hermes_cli/setup.py` returns nothing — the wizard configures models / providers / messaging platforms but never offers an "expose an HTTP API" step. So the **app must write these env vars itself** during bootstrap; we can't rely on the user picking them in the terminal wizard.
- **`18789` is a HermesAgentMobile invention.** In hermes-agent core the only mention is `hermes_cli/status.py:538`, where it's a hard-coded liveness probe (`sock.connect_ex(('127.0.0.1', 18789))`) — almost certainly OpenClaw-lineage cruft. We can keep `18789` (since the app already hard-codes it everywhere) or switch to the core default `8642`; the proposal recommends keeping `18789` to minimise churn in `GatewayService.kt` / `constants.dart` / the README.

**Implication for Part A:** the proposal's "one bootstrap step" is now firm (not a "possibly"). `bootstrap_service.dart` appends those four lines to `~/.hermes/.env` (generating a fresh `API_SERVER_KEY` once and persisting it), and the Flutter client reads the same key. Everything else in §3–4 stands as written. The verified API surface and observed event taxonomy are in §10.

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

- **Primary — `POST /v1/runs` → SSE `GET /v1/runs/{run_id}/events`.** hermes-agent is *agentic* (it calls tools), so the structured event stream is what makes a "proper" chat: you can render tool calls / tool results / "thinking" as distinct collapsible elements, and surface approval prompts inline. Body is `{"model":"hermes-agent","input":"<text>","session_id"?:"..."}` — note `"input"` (Responses-API-shape), **not** the OpenAI `{"messages":[...]}` shape (`POST /v1/runs` with a `messages` body returns `{"error":{"message":"Missing 'input' field"}}` — verified). Response is `{"run_id":"run_…","status":"started"}` (202). Then `GET /v1/runs/{run_id}/events` is an SSE stream of newline-delimited `data: {"event":"<name>","run_id":"…","timestamp":…,...}` records; the connection closes with `: stream closed`. `GET /v1/runs/{run_id}` returns `{"object":"hermes.run","run_id","status","created_at","updated_at","session_id","model","error?","last_event"}`.
- **Fallback — `POST /v1/chat/completions` with `stream:true`.** OpenAI-shape `{"model","messages":[…],"stream":true}` → `data: {"id":"chatcmpl-…","object":"chat.completion.chunk","choices":[{"index":0,"delta":{…},"finish_reason":…}], "usage"?:{…}}` then `data: [DONE]`. Token deltas only — no tool/thinking events. The client should feature-detect via `GET /v1/capabilities` (the `features` map advertises `run_submission`, `run_events_sse`, `chat_completions_streaming`, etc. — Phase 0 confirmed all are `true` in v0.13.0).

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
- **`flutter_app/lib/services/bootstrap_service.dart` (required — see §2.3 / §10).** Append a step that writes
  ```
  API_SERVER_ENABLED=true
  API_SERVER_PORT=18789
  API_SERVER_KEY=<generated once, persisted to SharedPreferences>
  GATEWAY_ALLOW_ALL_USERS=true
  ```
  to `~/.hermes/.env` (creating the file if absent, idempotent if these lines already exist — `hermes setup` writes other keys to the same file). No `config.yaml` edits needed; `python-dotenv` auto-loads it.

### 4.3 Behaviour

- **Send:** `POST /v1/runs` with `{"model":"hermes-agent","input":<user-text>,"session_id":<continuity-id>}` (the Responses-API body shape — `input`, *not* `messages`; see §3 / §10.4) + the `Authorization` and `X-Hermes-Session-Id` headers → `run_id` → open SSE `GET /v1/runs/{run_id}/events`. Fall back to `POST /v1/chat/completions` with `stream:true` and the OpenAI `{messages:[…]}` shape if `/v1/runs` ever proves unavailable.
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

1. ~~**The `:18789` question (Phase-0 gate).**~~ **Resolved by Phase 0** (§2.3, §10). The api_server adapter is *not* up by default; the bootstrap must write `API_SERVER_ENABLED=true / API_SERVER_PORT=18789 / API_SERVER_KEY=<generated> / GATEWAY_ALLOW_ALL_USERS=true` to `~/.hermes/.env` (auto-loaded by python-dotenv). One bootstrap step; confirmed working.
2. **hermes-agent API stability.** v`0.13.0` (HEAD `dd0923b`); pin the cloned commit/tag in the bootstrap so the API doesn't drift under the app. Their `dependencies` are exact-pinned (`pyproject.toml` comments cite the May-2026 supply-chain incident), which is reassuring for transitive stability — but the public route shape isn't versioned.
3. ~~**`/v1/runs` availability.**~~ **Confirmed available** in v0.13.0 (capabilities probe returned `run_submission: true`, `run_events_sse: true`, `run_stop: true`, `run_approval_response: true`, `tool_progress_events: true`, `approval_events: true`).
4. **No server-side chat history on the adapter** → client-side persistence; messages sent via the built-in terminal/TUI won't appear in the chat screen (acceptable — different surfaces).
5. **Auth.** Generating `API_SERVER_KEY` is now part of the bootstrap (item 1). The Flutter client reads the same key from `~/.hermes/.env` (or from a `SharedPreferences` mirror) and sends `Authorization: Bearer …`.
6. **Approval / sudo / secret flows on a small screen** — design the inline cards / modals carefully.
7. **Battery-optimization killing the gateway** — pre-existing failure mode, not introduced here, but the chat screen should surface "gateway not running" clearly.
8. **The optional Dashboard WebView needs a second process** (`hermes dashboard`) — extra moving part; that's why it's optional.
9. **`GATEWAY_ALLOW_ALL_USERS=true` weakens hermes-agent's user-allowlist** — fine on loopback-only Android (the proot user is effectively the device owner), but document the tradeoff. Without it the agent denied its own api_server runs in Phase 0.
10. **CI cache size** if a pre-baked rootfs is vendored for the fast-bootstrap test.

---

## 8. Phased roadmap

- ~~**Phase 0 — spike (GO / NO-GO).**~~ **Done.** See §2.3 + §10 — confirmed: as shipped the gateway opens zero ports; setting four env vars in `~/.hermes/.env` brings up the full `:18789` API; all advertised endpoints work; the `/v1/runs/*` SSE event taxonomy is documented in §10.
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

## 10. Appendix — Phase-0 spike transcript & references

**Setup:** clone `https://github.com/nousresearch/hermes-agent` @ `dd0923b` (v0.13.0); `python3 -m venv … && pip install -e hermes-agent && pip install aiohttp`. Run as a clean `HOME=/tmp/hermes-home` (no provider key configured).

### 10.1 As shipped — `python -m gateway.run` with **no** env vars

```
$ python -m gateway.run -v
WARNING __main__: No user allowlists configured. All unauthorized users will be denied.
WARNING __main__: No messaging platforms enabled.
$ ss -tln | grep -E '18789|8642|9119'   # → (nothing)
$ curl -m 2 http://127.0.0.1:18789/health → curl: (7) Failed to connect
$ curl -m 2 http://127.0.0.1:8642/health  → curl: (7) Failed to connect
$ curl -m 2 http://127.0.0.1:9119/         → curl: (7) Failed to connect
```
**The gateway opens zero listening ports.** The process stays alive (it ticks cron and is "running with 0 platforms"), but no HTTP surface is exposed. This is exactly the state the app boots into today.

### 10.2 With api_server enabled — write `~/.hermes/.env` then re-run

```
$ cat ~/.hermes/.env
API_SERVER_ENABLED=true
API_SERVER_PORT=18789
API_SERVER_KEY=testkey
GATEWAY_ALLOW_ALL_USERS=true

$ python -m gateway.run -v                 # logs (~/.hermes/logs/agent.log):
INFO __main__: Connecting to api_server...
INFO gateway.platforms.api_server: [Api_Server] API server listening on http://127.0.0.1:18789 (model: hermes-agent)
INFO __main__: ✓ api_server connected
INFO __main__: Gateway running with 1 platform(s)
```

`python-dotenv` (a core dep) auto-loads `~/.hermes/.env`. Same result as exporting the vars on the command line. The same file is where `hermes setup` already stores provider API keys, so the bootstrap can append these four lines without colliding.

### 10.3 Verified endpoints

```
$ curl -H 'Authorization: Bearer testkey' http://127.0.0.1:18789/v1/health
{"status":"ok","platform":"hermes-agent"}

$ curl http://127.0.0.1:18789/v1/models                              # no auth
{"error":{"message":"Invalid API key","type":"invalid_request_error","code":"invalid_api_key"}}

$ curl -H 'Authorization: Bearer testkey' http://127.0.0.1:18789/v1/models
{"object":"list","data":[{"id":"hermes-agent","object":"model","created":1778650893,
                          "owned_by":"hermes","permission":[],"root":"hermes-agent","parent":null}]}

$ curl -H 'Authorization: Bearer testkey' http://127.0.0.1:18789/v1/capabilities
{"object":"hermes.api_server.capabilities","platform":"hermes-agent","model":"hermes-agent",
 "auth":{"type":"bearer","required":true},
 "runtime":{"mode":"server_agent","tool_execution":"server","split_runtime":false, …},
 "features":{"chat_completions":true,"chat_completions_streaming":true,
             "responses_api":true,"responses_streaming":true,
             "run_submission":true,"run_status":true,"run_events_sse":true,
             "run_stop":true,"run_approval_response":true,
             "tool_progress_events":true,"approval_events":true,
             "session_continuity_header":"X-Hermes-Session-Id",
             "session_key_header":"X-Hermes-Session-Key","cors":false},
 "endpoints":{"health":{"method":"GET","path":"/health"},
              "health_detailed":{"method":"GET","path":"/health/detailed"},
              "models":{"method":"GET","path":"/v1/models"},
              "chat_completions":{"method":"POST","path":"/v1/chat/completions"},
              "responses":{"method":"POST","path":"/v1/responses"},
              "runs":{"method":"POST","path":"/v1/runs"},
              "run_status":{"method":"GET","path":"/v1/runs/{run_id}"},
              "run_events":{"method":"GET","path":"/v1/runs/{run_id}/events"},
              "run_approval":{"method":"POST","path":"/v1/runs/{run_id}/approval"},
              "run_stop":{"method":"POST","path":"/v1/runs/{run_id}/stop"}}}
```

`GET /v1/models` 200 served by `Server: Python/3.11 aiohttp/3.13.5` with `Content-Type: application/json; charset=utf-8`, `X-Content-Type-Options: nosniff`, `Referrer-Policy: no-referrer`.

### 10.4 `POST /v1/runs` body shape — *correction*

```
$ curl -X POST … -d '{"model":"hermes-agent","messages":[{"role":"user","content":"hi"}]}'
{"error":{"message":"Missing 'input' field","type":"invalid_request_error","param":null,"code":null}}

$ curl -X POST … -d '{"model":"hermes-agent","input":"hi"}'
{"run_id":"run_9b730378ae95414d9bbe24a71754a725","status":"started"}
```

`POST /v1/runs` uses the **Responses-API shape** (`{model, input, session_id?, …}`), **not** the Chat-Completions `{messages:[…]}` shape. The Dart client must send `input`. (`/v1/chat/completions` does still take `{messages}` — the two endpoints diverge here.)

### 10.5 `GET /v1/runs/{id}/events` — SSE wire format

```
$ curl -N -H 'Authorization: Bearer testkey' http://127.0.0.1:18789/v1/runs/run_9b73…/events
data: {"event":"run.failed","run_id":"run_9b73…","timestamp":1778650911.5763893,
       "error":"No inference provider configured. Run 'hermes model' to choose a provider …"}

: stream closed
```

Each event is a single `data: <json>\n\n` record. The JSON always has `event`, `run_id`, `timestamp`; per-event payload follows. The connection closes with a comment line `: stream closed`. (Failure happened here because no provider was configured — that's expected; once `hermes setup` writes e.g. `OPENROUTER_API_KEY` to `~/.hermes/.env`, runs would proceed and emit the documented `message` / `message.delta` / `tool.*` / `thinking` / `approval` / `status` events.)

`GET /v1/runs/{id}` returns `{"object":"hermes.run","run_id","status","created_at","updated_at","session_id","model","error?","last_event"}` — useful for polling and for displaying terminal-state errors after the SSE closes.

### 10.6 `POST /v1/chat/completions` (fallback) — SSE wire format

```
$ curl -N -X POST -H 'Authorization: Bearer testkey' -H 'Content-Type: application/json' \
       -d '{"model":"hermes-agent","stream":true,"messages":[{"role":"user","content":"hi"}]}' \
       http://127.0.0.1:18789/v1/chat/completions

data: {"id":"chatcmpl-35731c…","object":"chat.completion.chunk","created":…,"model":"hermes-agent",
       "choices":[{"index":0,"delta":{"role":"assistant"},"finish_reason":null}]}

data: {"id":"chatcmpl-35731c…","object":"chat.completion.chunk","created":…,"model":"hermes-agent",
       "choices":[{"index":0,"delta":{},"finish_reason":"stop"}],
       "usage":{"prompt_tokens":0,"completion_tokens":0,"total_tokens":0}}

data: [DONE]
```

Standard OpenAI streaming. No tool/thinking events — that's the cost of the fallback path. The non-streaming variant returns a structured error when no provider is configured (`{"error":{"message":"Internal server error: No inference provider configured. …","type":"server_error",…}}`).

### 10.7 References

- hermes-agent: <https://github.com/nousresearch/hermes-agent> (v0.13.0, HEAD `dd0923b`)
  - `gateway/run.py` (the daemon the app runs)
  - `gateway/platforms/api_server.py` (the `:18789` adapter — `DEFAULT_PORT = 8642`)
  - `gateway/config.py:1402-1426` (env-var → `Platform.API_SERVER` enablement)
  - `hermes_cli/setup.py` (no `api_server` mentions — confirmed)
  - `hermes_cli/status.py:538` (the lone `18789` reference in core — a liveness probe)
- Chatter-App: <https://github.com/ishandeveloper/Chatter-App>

### Decisions locked in with the owner

1. **Hermes-only — no piclaw pivot.** No Bun, no `armhf` drop, no rebrand. (The piclaw option and its Bun-coupling were explored separately and are out of scope here.)
2. **Native chat targets the `:18789` OpenAI-compatible REST/SSE** (`gateway/platforms/api_server.py`) — primary `POST /v1/runs` + SSE `GET /v1/runs/{id}/events`, fallback `POST /v1/chat/completions` `stream:true`. Not the `/api/ws` JSON-RPC (future upgrade path only).
3. **Chat UX reference = Chatter-App's layout**, reimplemented as fresh widgets in this app's theme.
4. **Dashboard WebView tab = optional / phase-later** (it needs a second `hermes dashboard` process).
5. **Termux-CLI path untouched.**
