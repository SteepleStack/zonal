# Release Notes - SPL Controller

## v2.7.0 — License Tiers, Feature Enforcement & Admin Portal CRUD

**Release Date:** June 4, 2026

### New Features

- **License tier system** — introduced Starter (3 zones / 1 controller), Pro (10 zones / 10 controllers), and Enterprise (unlimited) tiers with `TIER_CONFIGS` constants; trial licenses now carry full tier limits (time-gated only, no feature restrictions during trial)
- **Vendor-agnostic feature flag** — `qsys_integration` renamed to `audio_control` across all license data, defaults, and templates; prepares for future Dante/Yamaha integrations
- **Tier display in dashboard** — License Management panel now shows the license tier (Starter / Pro / Enterprise), Customer Portal button linking to `https://steeplestack.org/portal`, and hides activation/trial forms when a valid license is already active
- **`requireFeature` middleware** — new `makeFeatureHooks(license)` in the controller gates routes behind license feature flags; `audio_control` now enforced on MQTT publish (gain commands), settings write, and profile write routes; `web_dashboard` gates dashboard serving
- **Dashboard gain control gating** — gain floor/ceiling sliders, override engage/release buttons, and global Override All button are disabled with a license notice when `audio_control` is false
- **Admin portal editing** — `PATCH /admin/license/:key` endpoint allows updating customer name, max zones/controllers, expiration date, and features on existing licenses; admin portal now has tier preset buttons (Starter/Pro/Enterprise), feature checkboxes on license creation, and an inline edit panel per license row

### Bug Fixes

- Fixed pre-existing `ignoreDeprecations: "6.0"` TypeScript config error (invalid value — corrected to `"5.0"`)

### Infrastructure

- Docker image: `ghcr.io/steeplestack/zonal-controller:2.7.0`

---

## v2.6.0 — Security: Path Validation Hardening

**Release Date:** June 3, 2026

### Security

- **License path validation** — `LicenseClient` constructor now resolves and validates the license file path is contained within the data directory using `path.resolve` + `path.relative`; added full test suite covering traversal, encoded paths, and edge cases
- **MQTT TLS cert path validation** — `loadMqttTls` in server.ts now resolves and validates the `ca.crt` path is within the cert directory before reading

### Infrastructure

- Docker image: `ghcr.io/steeplestack/zonal-controller:2.6.0`

---

## v2.5.0 — Dashboard Redesign, Import/Export, Security Hardening

**Release Date:** June 3, 2026

### New Features

- **Steeplestack brand header** — header now shows the official Steeplestack logo mark and "Steeplestack / Zonal" title matching the website brand identity
- **Import/Export context menu** — the Export button on monitor and ganged cards is replaced with a single ⇅ button that opens a context menu with Import and Export options; importing accepts only `.json` files and sanitizes all values before applying to prevent XSS/injection
- **Version in settings dropdown** — current app version is now displayed in the user settings dropdown just above Sign Out

### Security

- **Path traversal protection in provision** — `walkDir` now validates all file paths are within the base directory using `path.resolve` + `path.relative` before reading; added full test suite covering traversal, symlinks, and encoded paths
- **SAST fixes** — resolved Aikido findings: pinned `softprops/action-gh-release` and `codecov/codecov-action` to commit hashes; fixed template injection in mirror workflow; added non-root user to test Dockerfiles; moved hardcoded license key to env var; fixed `baseUrl` TS deprecation; fixed shell injection in git hook
- **Dependency CVE patches** — `brace-expansion` v5 pinned to `>=5.0.6` (CVE-2026-45149); `ws` pinned to `>=8.20.1`; `postcss` pinned to `>=8.5.10`

### Infrastructure

- Docker image: `ghcr.io/steeplestack/zonal-controller:2.5.0`
- Test stack updated to use GHCR image

---

## v2.4.3 — Fix: brace-expansion Override Scope

**Release Date:** June 3, 2026

### Bug Fix

- **brace-expansion override** — scoped the npm override to `brace-expansion@>=5.0.0` to avoid forcibly upgrading v1.x/v2.x consumers (nx internals) to v5, which has an incompatible module API. The v5 CVE fix (CVE-2026-45149) is still applied.

---

## v2.4.2 — Security: Dependency CVE Patches

**Release Date:** June 3, 2026

### Security

Patched all high and medium severity CVEs identified by Docker Scout in the v2.4.1 image.

- **fastify** upgraded 5.2.2 → 5.8.5 (fixes CVE-2026-25223, CVE-2025-32442, CVE-2026-3635 — Content-Type validation bypass)
- **next** upgraded 15.5.16 → 15.5.19 (fixes CVE-2026-45109 — authentication bypass)
- **brace-expansion** pinned ≥ 5.0.6 via overrides (fixes CVE-2026-45149 — ReDoS)
- **ws** pinned ≥ 8.20.1 via overrides (fixes CVE-2026-45736)
- **postcss** pinned ≥ 8.5.10 via overrides (fixes CVE-2026-41305 — XSS)

No functional changes. Remaining unfixed: `busybox` CVE-2025-60876 (no fix available in Alpine 3.23).

---

## v2.4.1 — Bug Fixes: Monitor Buffering, Provisioner CRLF, GHCR Registry

**Release Date:** June 3, 2026

### Bug Fixes

- **monitor-cpp** — add `std::unitbuf` to flush stdout immediately when piped to journald; previously logs would buffer silently until process termination
- **provision-template.ps1** — base64-encode the remote shell script before piping to SSH; PowerShell adds `\r\n` line endings on Windows, causing `bash: set: pipefail\r: invalid option` failures on the Pi
- **docker-compose.deploy.yml** — switch `lounge-controller` image reference from Docker Hub to GHCR (`ghcr.io/steeplestack/zonal-controller`)
- **server cert** — regenerated mosquitto TLS server certificate with SAN entries for `spl.elc-server.com` and `172.16.0.111`; libmosquitto performs strict hostname verification and was rejecting the CN-only cert

---

## v2.4.0 — C++ Monitor, License Server, Hardware Library

**Release Date:** June 2, 2026

### New Feature — C++ SPL Monitor (Raspberry Pi)

A native C++ implementation of the Pi SPL monitor replaces the Node.js monitor for resource-constrained deployments. The C++ monitor uses a fraction of the CPU and memory of the Node version, making it suitable for older Pi hardware and deployments where multiple monitors share a single board.

- **ALSA audio capture** — reads directly from ALSA PCM with auto-detection of UMIK/miniDSP USB microphones (`AUDIO_DEVICE_INDEX=-1`). No PortAudio dependency.
- **DSP pipeline** — A-weighting filter (bilinear-transform IIR, matched to the Node implementation), RMS accumulation, Leq averaging over a configurable interval, and a per-unit calibration trim (`EXTRA_TRIM_DB`).
- **mTLS MQTT** — publishes over TLS 1.2+ with the same client certificate scheme and topic schema as the Node monitor. Fully compatible with existing broker ACLs and controller gang logic.
- **`.env` configuration** — identical config surface to the Node monitor. Existing provision scripts and packages are compatible without changes.
- **TEST\_MODE** — generates synthetic SPL readings (configurable base, swing, and period) when `TEST_MODE=true`. No microphone required for CI or staging validation.
- **CMake build** — `cmake -B build -DCMAKE_BUILD_TYPE=Release && cmake --build build -j$(nproc)` produces a single static-linked binary. Cross-compilation to ARMv7/ARM64 supported via CMake toolchain files.
- **`provision-pi-cpp.ps1`** — Windows PowerShell provisioner that copies source to the Pi over SSH, builds with CMake on-device, writes the `.env`, installs mTLS certs, and registers a systemd unit. Mirrors `provision-pi.ps1` for the Node monitor; no WSL required.
- **`test/Dockerfile.monitor-cpp`** — Ubuntu 24.04 multi-stage image (builder + runtime) for CI validation of the C++ monitor build and `TEST_MODE` output.

### New Feature — License Server

A production-grade license server backs the Zonal licensing system. Deployed as a Docker container behind Cloudflare Access.

- **License activation and validation** — Zonal controllers verify their license key on startup. Keys are bound to the activating machine on first use.
- **Admin portal** — Browser-accessible management interface for creating, revoking, and reviewing licenses.
- **CI/CD pipeline** — Automatically built and deployed on every update.

### New Feature — Release Mirror Workflow

`.github/workflows/mirror-release.yml` automatically mirrors every published release from this private dev repo to the public-facing `steeplestack/zonal` repository. On release publish, it downloads all release assets, copies the release title and body, and re-publishes with the same tag. `RELEASE_NOTES.md` is also pushed directly to the public repo via the GitHub Contents API.

### Hardware

- **Silvertel AG9205-S KiCad library** — schematic symbol and PCB footprint (SIL-10, 2.54 mm pitch) for the PoE module used in the SPL Monitor hardware reference design. Added to `hardware/kicad-lib/Silvertel/`.

---

## v2.3.1 — Bug Fixes, UX Polish, Security

**Release Date:** May 19, 2026

### Bug Fixes

- **Dashboard bridge not started** — `dashboardMqtt.start()` was dropped during a merge, causing the SSE snapshot to always be empty and monitors to never appear after a container restart.
- **Dry Run state not persisted** — `onConfigUpdate` updated the runtime `dryRun` flag but not `config.dry_run`, so every controller ack reset the UI back to off. Both are now synced and saved to SQLite.
- **Gang card state sync** — Controller config/override/schedule acks now distribute to ALL monitors sharing a controller ID, fixing state desync on ganged cards.
- **Profiles backend missing** — `/api/controllers/:id/profiles` endpoints were not registered. Added `controller_profiles` SQLite table, `AuthManager` profile methods, and the route.
- **mosquitto.acl replaced by directory** — Docker created the ACL bind-mount path as a directory when the source file was absent. File restored.
- **build.sh conflict markers** — Leftover merge conflict markers in `build/build.sh` prevented platform builds.

### Security

- **Non-root Docker container** — Controller image now runs as the built-in `node` user (uid 1000). An `su-exec` entrypoint transfers `/data` volume ownership at startup then drops root before starting Node.

### UX / Dashboard

- **Profile Load applies immediately** — Clicking Load publishes settings directly to the controller. No tab switching or separate Save needed.
- **Profile Load toast** — Green toast confirms: `"Profile Name" loaded to "controller-id"`.
- **Profile Update always enabled** — No longer requires pending local changes; any time is valid for snapshotting live settings into a profile.
- **Dry Run latch button** — Replaced the checkbox with a full-width latch button that engages/disengages instantly (no Save required). Active state shows amber styling and "⏸ Dry Run ON".
- **Tab badge bubbles no longer clipped** — Badges moved from `position: absolute` to `display: inline-flex` so they no longer clip inside the scrollable tab bar on mobile.
- **Mobile card centering** — Cards are centred on narrow viewports (`min(340px, 100%)` width, `justify-content: center` on the row, reduced padding below 400px).
- **SSH single session** — `provision-pi.sh` uses SSH ControlMaster (one password prompt for the entire run). `provision-pi.ps1` stages all files locally then does one `scp -r` + one `ssh`.

### Cleanup

- Removed all stale Python artefacts: `.venv/`, `tests/`, `build/build_app.py`, `build/pyinstaller/`, all `__pycache__/` directories.
- Removed old Python `license-server/` (Flask/Gunicorn); `apps/license-server/` (Node.js/Fastify) is authoritative.
- Trimmed Python boilerplate from `.gitignore`.

---

## v2.3.0 — mTLS Security, Monitor Ganging, Provision Packages

**Release Date:** May 14, 2026

### Security — mTLS MQTT Authentication

- **Mutual TLS replaces username/password** — The MQTT broker now requires client certificates instead of a shared password. All services (`spl-controller`, `spl-monitor`, `spl-dashboard`) authenticate with unique client certificates signed by a private CA. No `MQTT_PASSWORD` is needed anywhere in the stack.
- **Auto-generated certificates** — The `mosquitto-init` container generates the entire PKI (CA + server cert + per-role client certs) on first `docker compose up` using OpenSSL. Certs are written to `./certs/` and reused on subsequent restarts. No user action required.
- **Port change: 1883 → 8883** — The broker now listens on the standard MQTTS port. WebSocket stays on 9001 (now TLS). Existing deployments must update `MQTT_PORT` to `8883`.
- **Pi monitors use mTLS** — The provision script (bash and PowerShell) copies `ca.crt`, `spl-monitor.crt`, and `spl-monitor.key` to the Pi automatically. No passwords on the Pi either.
- **Controller/dashboard bridge full-cert TLS** — Both MQTT clients inside the controller use `rejectUnauthorized: true` with hostname verification (SANs include `mosquitto`). The Pi monitor uses `checkServerIdentity: () => undefined` (cert chain still verified) to support arbitrary broker IPs.

### New Feature — Monitor Ganging

- **Gang multiple monitors into one controller** — A new "Gang" tab on each zone card lets you select any combination of online monitors. The controller averages their SPL readings before applying gain control — ideal for large zones needing multi-point coverage with a single Q-SYS gain.
- **Graceful degradation** — If a ganged monitor goes offline, its readings age out after 30 seconds and the remaining monitors continue to drive control without interruption.
- **GANG badge** — When a gang of 2+ monitors is active, the zone card shows a `GANG N` header badge. The Gang tab also shows a count badge.
- **Configurable per controller** — Gang membership is stored in controller settings (SQLite), published via MQTT, and survives restarts. Clearing the gang returns the controller to accept-all mode.

### New Feature — Provision Package Download

- **Dashboard generates a self-contained ZIP** — The "Add Monitor" modal now has a **Download Package** button. It downloads a ZIP containing the built monitor app, mTLS certificates, pre-configured `.env`, `setup.sh` (remote install script), `provision-pi.sh` (bash), and `provision-pi.ps1` (PowerShell). No repo clone or extra tools required.
- **Pi address baked in** — All values including the Pi SSH address are pre-filled in the downloaded scripts. Users unzip and run one command.
- **UMIK-1 calibration file upload** — Upload a UMIK-1 `.cal` file in the modal; it is embedded in the package and written to the Pi during provisioning.
- **SSH auth options** — The Gang tab shows clear instructions for key-based auth (recommended) and password auth (via `sshpass` on Mac/Linux). Windows: `$env:SSH_KEY` points to a key file.

### New Feature — Native Windows PowerShell Provisioning

- **No WSL required** — `provision-pi.ps1` uses Windows built-in OpenSSH (`ssh`, `scp`) available since Windows 10 1809. No Git Bash, no Cygwin, no WSL.
- **PowerShell 5.1 compatible** — Template uses ASCII-only strings (no em-dash or box-drawing characters) and a UTF-8 BOM to avoid Windows-1252 encoding misparse. Avoids `&&` in SSH arguments (split into separate calls).
- **`setup.sh` separated** — The remote Pi setup logic lives in its own `setup.sh` in the package; both the bash and PowerShell provisioners SCP it to the Pi and run it, eliminating PowerShell heredoc complexity.

### Dashboard — Remove Monitor

- **Trash button on monitor cards** — Hovering a zone card reveals a trash icon. First click shows an inline "Remove / ✕" confirmation; confirming publishes empty retained messages to `church/monitors/hello/{id}` and `church/{id}/config/monitor/ack`, clearing the broker and making the card vanish instantly for all connected browsers.
- **Real-time SSE push** — The `'removed'` event propagates to all connected dashboard sessions via SSE; no refresh needed.

### Dashboard — Add Monitor Modal Overhaul

- **Pi address field** — The modal now has a dedicated "Pi Address" field (`admin@192.168.1.50`) baked into the downloaded provision package.
- **Removed WSL dependency** — Windows tab no longer shows a `wsl bash -lc` command. The PowerShell command uses native `& ".\provision-pi.ps1"`.
- **Port updated** — Status strip now shows `:8883` (MQTTS) instead of `:1883`.
- **Preflight updated** — Removed "MQTT password is set" row; replaced with "certs are generated automatically."

### Standalone (Non-Docker) Broker Fix

- **Localhost-only anonymous mode** — When the controller starts its own Mosquitto instance (no external broker reachable), it now writes a minimal config binding to `127.0.0.1` with `allow_anonymous true`. This prevents connection failures on standalone installs where there is no network threat.

### Bug Fixes

- React error #185 (infinite render loop) — `useMonitorsStore(s => Object.values(s.monitors))` created a new array reference every render; fixed by selecting `s.monitors` (stable reference) and calling `Object.values` outside the selector.
- `provision-pi.sh` now passes `SSH_ARGS` and `SSH_PREFIX` to all `ssh`/`scp` calls, enabling key and password auth throughout the full provisioning flow.
- `ganged_monitor_ids` is cleared from `gangReadings` on config change so stale readings from a previous gang configuration cannot bleed into a new one.

---

## v2.2.1 — MQTT Reliability, Pi Provisioning, Release Hardening

**Release Date:** May 13, 2026

### MQTT / Deployment

- **Generated Mosquitto credentials** — Docker Compose now creates `spl-controller`, `spl-monitor`, and `spl-dashboard` broker users from one shared `MQTT_PASSWORD`, preventing stale password-file drift between controller, broker, dashboard, and Pi monitors.
- **Authenticated broker defaults** — Mosquitto examples now require authentication for TCP and WebSocket listeners and mount the generated password file from a Docker volume.
- **Restart-safe stack** — Controller and Mosquitto examples keep `restart: unless-stopped`; Pi monitor services are installed as enabled systemd units with `Restart=always`.
- **Consumer Docker example** — Added a consumer-facing Docker README and `.env.example` for the controller/Mosquitto stack.

### Pi Monitor Setup

- **Provisioner verifies MQTT from the Pi** — `consumer/monitor/provision-pi.sh` now performs a real MQTT login/publish check from the Raspberry Pi before reporting success.
- **Safer `.env` writing** — Pi provisioning writes a quoted env file locally, copies it to the Pi, and avoids shell-expanding passwords or display names.
- **Runtime dependencies installed explicitly** — Provisioning installs `mqtt` and `dotenv` on the Pi so a clean monitor deployment does not rely on stale `node_modules`.
- **Credential-aware monitor bundle** — The monitor app requires `MQTT_USERNAME` and `MQTT_PASSWORD` and passes both into `mqtt.connect`.

### Dashboard

- **Monitor cards survive browser refresh** — The controller MQTT bridge caches the latest non-log MQTT state and replays it to new dashboard SSE connections, so retained monitor presence/config/SPL state appears immediately after refresh.
- **Add Monitor modal updated** — The modal now avoids hardcoded customer IPs, uses editable broker host detection, carries the display name into provisioning, and explains the new preflight/restart behavior.

### Release / Public Repo Hygiene

- **Generic public defaults** — Removed customer-specific IPs, license examples, and site details from public examples and docs.
- **Docker image namespace** — Build scripts and compose examples now use `steeplestack/zonal-controller`.
- **Lockfile release support** — `package-lock.json` is included for reproducible Docker image builds.

---

## v2.2.0 — Dashboard UX, User Management, Settings Import

**Release Date:** May 11, 2026

### New Features

- **Settings import** — Export your zone settings to JSON and import them on any zone. The Import button sits next to Export in every zone's Settings tab; it restores both threshold/gain settings and the full schedule, marking them dirty for review before saving.

- **Create and edit users** — The Admin panel Users tab now has a full "Add User" form (username, password, role assignment) and per-user "Edit" inline form to change passwords and reassign roles without leaving the page. Enable/Disable and Delete were already there.

- **Create and edit roles** — The Admin panel Roles tab now has an "Add Role" form and per-role "Edit" inline form. Both show a permission grid with all available permissions as checkboxes, making it easy to compose custom roles without knowing permission names.

- **OS-aware monitor setup command** — The Add Monitor dialog now has a Mac/Linux and Windows toggle. Mac users see the standard `bash` command; Windows users see the `wsl bash` variant with an inline explanation of how to enable WSL if it isn't installed yet.

### Improvements

- **Consumer-friendly language in Add Monitor dialog** — Removed developer jargon: "dev machine" is now "your computer", "repo root" is gone, "Git Bash / WSL" is replaced with plain OS-specific step instructions, and the MQTT host hint is clearer.

- **MQTT_URL override for monitor** — The monitor app now accepts a `MQTT_URL` environment variable so it can connect via WebSocket (`ws://`) or any other URL scheme, independent of `MQTT_HOST` and `MQTT_PORT`. Fixes connectivity on systems where port 1883 is taken by a pre-installed broker.

### UX / Accessibility (impeccable audit)

- **Contrast fixed across all panels** — `--muted` raised to `oklch(54%)`, giving ≥ 4.5:1 contrast on all panel backgrounds. Previously failing WCAG AA throughout.
- **Focus ring** — Universal `:focus-visible` ring added (`var(--accent2)`, 2 px). All buttons and controls now have a visible keyboard indicator.
- **Reduced motion support** — `@media (prefers-reduced-motion: reduce)` guard added; all animations and transitions cut to near-zero for users who have opted out.
- **Color-only status fixed** — Connection bar collapsed state now shows a text label ("Connected / Connecting / Offline") alongside the status dot.
- **Invisible text fixed** — `lastSeen` timestamp, EventLog count badge, log timestamps, and empty-state messages were using `var(--border2)` (near-invisible on dark). Fixed to readable values.
- **9 px text raised** — Section group labels, sparkline label, EventLog title bumped to 10–11 px — readable in dim ambient light.
- **CSS bugs fixed** — Duplicate `.panelTitle` class collision in MonitorPanels (second declaration silently overrode the first), orphaned `--thumb-r` custom property declared outside `:root`, and em-dash in "— EVENT LOG" title (replaced with "EVENT LOG").
- **No more native `confirm()` dialog** — Settings tab no longer blocks tab switching with a browser confirm dialog; the dirty-state dot indicator conveys unsaved changes without interrupting navigation.

### Security

- **fastify `5.2.2` → `5.8.5`** — 4 CVEs patched (2 HIGH, 1 MEDIUM, 1 LOW)
- **next `15.2.9` → `15.5.18`** — 7 CVEs patched (1 HIGH, 6 MEDIUM)
- Transitive dependencies `ip-address`, `postcss`, `@babel/runtime` also updated

### Bug Fixes

- `createUser` username lookup used the original un-normalised username for the final ID fetch — would fail to return the new user's ID for any uppercase input
- `applyGain` / `adjustGain` were called without `await` or `void`, allowing concurrent Q-SYS writes to race. `gainPending` guard added.
- `TEST_MODE` monitor interval was fixed at startup and ignored hot-config changes to `publish_interval_sec` — replaced `setInterval` with self-rescheduling `setTimeout`
- `APP_VERSION` was hardcoded `'2.1.0'`; now read from `package.json` at startup with a two-path probe that works both in Docker and in dev
- Dead `monitor/src/main.ts` (contained only `console.log('Hello World')`) deleted — build entry was already `index.ts`
- Unused `pg` / `@types/pg` dependencies removed

---

## v2.1.1 — Patch: Bug fixes, security updates, code quality

**Release Date:** May 9, 2026

### 🔒 Security

- **fastify `5.2.2` → `5.8.5`** — patches 4 CVEs (2 HIGH, 1 MEDIUM, 1 LOW):
  - HIGH CVE-2026-25223 — interpretation conflict
  - HIGH CVE-2025-32442 — improper type validation
  - MEDIUM CVE-2026-3635 — use of less-trusted source
  - LOW CVE-2026-25224 — unbounded resource allocation
- **next `15.2.9` → `15.5.18`** — patches 7 CVEs (1 HIGH, 6 MEDIUM):
  - HIGH GHSA-q4gf-8mx6-v5v3 — DoS via resource exhaustion
  - MEDIUM CVE-2025-57822 — SSRF
  - MEDIUM CVE-2026-29057 — HTTP request smuggling
  - MEDIUM CVE-2025-57752 — sensitive data in cache
  - MEDIUM CVE-2025-55173 — input validation bypass
  - MEDIUM CVE-2025-59471, CVE-2026-27980 — resource exhaustion
- **Removed unused `pg` / `@types/pg`** — dead dependencies eliminated

### 🐛 Bug Fixes

- **`createUser` username lookup** — user IDs were looked up using the original (un-normalised) username after storing as lowercase; any call with uppercase letters silently failed to return the new user's ID
- **Concurrent gain operations** — `applyGain` and `adjustGain` were called without `await` or `void`, allowing overlapping Q-SYS writes to race. Added `gainPending` guard and explicit `void` on all fire-and-forget callsites
- **TEST_MODE interval ignored hot-config** — `setInterval` captured the initial `publish_interval_sec` at startup; MQTT config updates had no effect. Replaced with a self-rescheduling `setTimeout` that reads `cfg.publish_interval_sec` on each tick

### ⚙️ Improvements

- **Scoped log forwarding** — replaced global `console` monkey-patching in `SplController` with a private `log()` method that calls the real console and also publishes to the MQTT log topic. Eliminates the risk of patching third-party library output and removes the reentrancy guard complexity
- **Audio buffer allocation** — monitor's audio capture loop was calling `Buffer.concat` on every PortAudio chunk (hundreds/sec at 48 kHz). Chunks now accumulate in an array and are concatenated once per SPL interval
- **`APP_VERSION` sourced from `package.json`** — was hardcoded `'2.1.0'`; now read at startup via `readFileSync` so it cannot drift from the package version
- **Removed dead `monitor/src/main.ts`** — contained only `console.log('Hello World')`; the actual build entry is `index.ts` (unchanged in `project.json`)

---

## v2.0.1 — Patch: Native module fix, binary renaming, node:sqlite

**Release Date:** May 8, 2026

### 🐛 Bug Fixes

- **Fixed native module ABI crash on startup** — replaced `better-sqlite3` (compiled native addon) with Node.js built-in `node:sqlite` (`DatabaseSync`). Eliminates the `NODE_MODULE_VERSION` mismatch error when running the packaged executable.
- **Fixed `@fastify/autoload` directory scan crash** — removed unused Nx scaffold code (`app/`) that caused pkg to fail with a missing directory error at startup.
- **Fixed `main.ts` import** — removed leftover Nx-generated entry point that referenced the deleted `app/app` module.
- **Fixed `tsconfig.app.json` `moduleResolution`** — re-applied `node` override after regression.

### 📦 Renamed Binaries

All binaries now use the product name **Steeplestack Zonal Controller**:

| Platform | Binary |
|----------|--------|
| Windows | `Steeplestack-Zonal-Controller.exe` |
| Linux binary | `Steeplestack-Zonal-Controller-linux` |
| Linux DEB | `steeplestack-zonal-controller_2.0.1_amd64.deb` |
| Docker | `nordadmin/steeplestack-zonal-controller:2.0.1` |

### ⚙️ Technical

- Executable target updated from `node20` → `node22` (matches `node:sqlite` availability)
- `node:sqlite` transactions use explicit `BEGIN`/`COMMIT`/`ROLLBACK` (no `.transaction()` wrapper needed)
- `dist/` cleaned of all v1.x legacy artifacts

---

## v2.0.0 — Full TypeScript / Node.js Rewrite

**Release Date:** May 7, 2026

### 🚨 Breaking Changes

- **Python runtime eliminated** — the application no longer requires Python. All components are now TypeScript/Node.js.
- **PyInstaller replaced by `pkg`** — desktop executables are now built with `@yao-pkg/pkg` (Node.js bundler).
- **New project structure** — codebase is now an Nx monorepo with `apps/` containing `dashboard`, `controller`, `license-server`, and `monitor`.
- **Config directory unchanged** — existing config, auth.db, and license files at `~/.steeplestack/spl-controller/` continue to work.

### 🏗️ Architecture

Complete rewrite of all components to TypeScript/Node.js:

| Component | Before | After |
|-----------|--------|-------|
| Dashboard | Vanilla HTML + inline JS (5,400 lines) | Next.js 14, App Router, React, CSS Modules |
| Controller HTTP server | Python `http.server` | Fastify (TypeScript) |
| Auth/RBAC | Python SQLite | TypeScript + `better-sqlite3` (same schema) |
| License client | Python `requests` | TypeScript `fetch` |
| Controller daemon | Python MQTT + TCP | TypeScript `mqtt` + Node.js `net` |
| Pi monitor | Python PyAudio + scipy | Node.js `naudiodon` + custom DSP |
| State management | Global JavaScript variables | Zustand stores |
| Build tooling | PyInstaller | Nx + `pkg` |

### 🎨 Dashboard Rewrite (Next.js)

- **React components** — 14 components replacing a single monolithic HTML file
- **Zustand state** — `mqttStore`, `monitorsStore`, `authStore`, `uiStore`
- **CSS Modules** — same dark green aesthetic, properly modularised
- **TypeScript API client** — fully typed, all `/api/*` endpoints
- **`useMqtt` hook** — MQTT connect/subscribe/publish abstracted into a React hook
- **Static export** — `next build` generates flat HTML/JS for Node.js to serve

### 🔧 Nx Monorepo

```
apps/
├── dashboard/        ← Next.js 14 (static export)
├── controller/       ← Fastify backend + MQTT + Q-SYS
└── monitor/          ← Node.js Pi monitor (naudiodon)
```

Dev workflow:
```bash
NEXT_PUBLIC_API_BASE=http://localhost:8080 npx nx serve dashboard
npx nx serve controller
```

### 📦 Build & Packaging

- **Windows/Linux/macOS executables** — `pkg` bundles Node.js runtime + app into a single binary (~50 MB)
- **Docker** — `node:20-alpine`, no Python dependency
- **Linux DEB** — built from the `pkg` Linux binary via `dpkg-deb`
- **Unified build script** — `bash build/build.sh --platform all --docker`

### 🐛 Bug Fixes & Improvements

- All Python `apply_config` silent error swallowing fixed at the TypeScript layer
- A-weighting DSP ported exactly from scipy (bilinear transform, IIR filter — same math)
- `naudiodon` type shim included for development on non-Pi platforms
- Dashboard `moduleResolution` conflicts resolved in all `tsconfig.app.json` files

### 📋 Upgrade Notes

1. No data migration needed — existing config, auth database, and license files carry over automatically.
2. Docker: update `docker-compose.deploy.yml` to reference `nordadmin/steeplestack-zonal-controller:2.0.0`
3. Pi monitors: the updated monitor package replaces `spl-monitor.py`; requires `naudiodon` and PortAudio on the Pi (`sudo apt install portaudio19-dev && npm install naudiodon`)

---

## v1.1.0

**Release Date:** May 7, 2026

### 🔐 Authentication & Role-Based Access Control

- **User accounts** — create, edit, enable/disable, and delete users from the web dashboard
- **Custom roles** — define roles with granular permission sets; assign multiple roles per user
- **Permissions** — `view_dashboard`, `adjust_gain`, `set_override`, `manage_monitors`, `manage_config`, `manage_users`, `manage_roles`, `manage_license`
- **Permission-gated UI** — override, dry run, schedule, controller settings, and monitor config sections are hidden based on the logged-in user's permissions
- **Login page** — secure sign-in with session cookies (HttpOnly, SameSite=Strict)
- **Lock screen** — `Ctrl+Shift+L` or 🔒 button locks the dashboard; lock state is server-side so page refresh cannot bypass it
- **Admin panel** — full user and role management UI accessible from the header dropdown

### 🔑 License Management

- **License panel in dashboard** — view current license status, activate a commercial key, start a trial, and refresh validation — all from within the UI
- **Headless activation** — `SPL_LICENSE_KEY` + `SPL_LICENSE_EMAIL` environment variables bypass the interactive wizard for Docker/CI deployments

### 🐳 Docker & Multi-Zone Deployment

- **Official Docker image** — `nordadmin/steeplestack-zonal-controller:latest` on Docker Hub
- **Multi-zone architecture** — one primary container (dashboard + controller) plus one container per additional zone using `--no-dashboard` flag; all share one Mosquitto broker
- **`CONTROLLER_ID` + `QSYS_COMPONENT` env vars** — configure each zone independently without touching config files
- **Docker Compose deploy file** — `consumer/platforms/docker/docker-compose.deploy.yml` with commented zone templates
- **CVE hardening** — multi-stage Docker build removes `gcc`/`binutils` (eliminated 54 CVEs); base image upgraded to `python:3.13-slim`

### 🖥️ Platform Improvements

- **Windows system tray** — right-click menu with Open Dashboard and Quit; console hides after first-run setup
- **macOS tray** — pystray AppKit backend enabled
- **Linux headless** — tray gracefully skipped when no `$DISPLAY`/`$WAYLAND_DISPLAY`; credentials written to `first_run_credentials.txt`
- **First-run admin credentials** — shown as a native dialog (Windows MessageBox / macOS osascript) after license activation; no longer disappears before you can read it
- **`--no-dashboard` flag** — run controller-only mode for multi-zone Docker deployments
- **`--no-mqtt` flag** — skip embedded Mosquitto when an external broker is available

### ⚙️ Controller & Settings

- **Per-controller settings in SQLite** — thresholds, gain limits, and tuning parameters are now stored per controller ID in the auth database, not in `config.json`
- **Dashboard saves settings on apply** — clicking Apply in the controller settings panel persists to the server via `PUT /api/controllers/<id>/settings`; settings survive restarts
- **Environment variable overrides** — `MQTT_HOST`, `MQTT_PORT`, `QSYS_IP`, etc. override config file values for Docker/server deployments

### 🏗️ Infrastructure & Codebase

- **Codebase reorganised** — clean monorepo structure:
  - `consumer/` — everything shipped to end users (core, dashboard, monitor, platforms)
  - `license-server/` — SteepleStack-internal licensing infrastructure
  - `build/` — build tooling (PyInstaller, Docker build container, assets)
  - `docs/` — documentation
- **Automated test suite** — pytest unit tests (auth, license, controller logic), integration tests (all `/api/*` endpoints, license server with Docker Postgres), and Playwright E2E tests (login, lock screen, RBAC, admin panel)
- **CI workflow** — GitHub Actions runs tests on PR and version tags
- **Linux dev container** — `build/docker-compose.linux.yml` for reproducible Linux/DEB builds on any OS
- **Service worker fixed** — API calls (`/api/*`) and auth routes are never cached; dashboard HTML uses network-first strategy

### 🐛 Bug Fixes

- Fixed `get_resource_path()` returning wrong path on Linux/Docker (was going up two directories instead of one)
- Fixed admin panel not opening due to JavaScript TDZ error caused by `getElementById` on elements parsed after the script tag
- Fixed `wheel` HIGH CVE (CVE-2026-24049) and `pip` CVEs in Docker image
- Fixed 37-second shutdown hang when browser has an open connection to the dashboard server
- Fixed console window hiding before license wizard could run, leaving the app as a zombie process
- Fixed Mosquitto trying to start on every launch even when already running (now checks port 1883 first)

### 📦 What's Included

- **Windows**: `SteepleStack-Zonal-Controller.exe` (UAC-elevated, console hidden after first run)
- **Linux**: `spl-controller_1.0.0_amd64.deb`
- **Docker**: `nordadmin/steeplestack-zonal-controller:latest`
- **macOS**: `.app` bundle (build via `consumer/platforms/macos/build.sh`)

---

## v1.0.0 (Production Release)

**Release Date:** May 1, 2026

### 🎉 Major Features

- **Cross-Platform Support**: Native executables for Windows, macOS, and Linux
- **Commercial Licensing**: Built-in license activation, remote validation, and revocation support
- **License Server Admin Portal**: Web interface for creating, listing, revoking licenses, and debugging
- **Automatic License Management**: Periodic remote refresh with graceful shutdown on revocation/expiration
- **Manual License Tools**: CLI commands for license status (`--license-status`) and manual refresh (`--refresh-license`)
- **Embedded MQTT Broker**: Built-in Mosquitto for local messaging infrastructure
- **Headless Linux Support**: Configuration wizard and systemd daemon/service support for server deployments
- **Q-SYS Integration**: Robust controller process orchestration with audio system integration
- **Production Packaging**: Linux `.deb` artifacts with comprehensive installer guidance
- **Reliability Improvements**: Enhanced license datetime parsing and refresh diagnostics

### 📦 What's Included

- **Windows**: `steeplestack-zonal-controller.exe` installer
- **macOS**: `steeplestack-zonal-controller.dmg` disk image
- **Linux**: `steeplestack-zonal-controller_1.0.0_amd64.deb` package
- **Documentation**: Complete deployment and operations guide

### 🔧 Technical Improvements

- Fixed ISO8601 datetime parsing for license validation
- Improved error handling in license refresh operations
- Enhanced startup diagnostics and logging
- Robust subprocess management for controller orchestration
- Cross-platform build system with PyInstaller

### 🚀 Getting Started

1. **Download** the appropriate installer for your platform
2. **Install** using the platform-specific instructions
3. **Configure** Q-SYS connection and license activation
4. **Deploy** Raspberry Pi monitors using the provision script
5. **Monitor** via web dashboard at `http://localhost:8080`

### 📋 System Requirements

- **OS**: Windows 10+, macOS 10.14+, Ubuntu 20.04+
- **RAM**: 4 GB minimum (8 GB recommended)
- **Storage**: 500 MB free space
- **Network**: LAN access to Q-SYS and Raspberry Pi monitors

### 🔒 Security Notes

- License keys are validated against remote server
- Machine binding prevents unauthorized license sharing
- Admin portal requires authentication token
- All network communications use standard protocols

### 🐛 Known Issues

- Q-SYS connection may require manual IP configuration in some networks
- macOS may prompt for accessibility permissions on first run
- Linux service requires systemd for auto-start functionality

### 📚 Documentation

- [Production Deployment Guide](PRODUCTION_DEPLOYMENT.md)
- [Build Guide](BUILD_GUIDE.md)
- [Quick Build](QUICK_BUILD.md)
- [API Documentation](https://docs.steeplestack.org)

### 🙏 Acknowledgments

Special thanks to the beta testers and early adopters who helped validate this production release.

---

**Previous Versions**
- No previous public releases

**Support**: support@steeplestack.org
**Issues**: https://github.com/SteepleStack/zonal/issues
**Community**: https://community.steeplestack.org