# Specification: Native Antigravity CLI (`agy`) Support in Omarchy

## Problem Statement

Users who adopt Google's Antigravity agent CLI (`agy`) on Omarchy have no first-class system support. Currently, users must configure a manual CLI shim under `~/.local/bin/gemini` that intercepts calls to legacy Gemini and rewrites arguments to target `agy`. Because `agy` is not recognized as an official provider by the desktop shell, its sessions, prompt activity, token usage, and subscription tier never appear in the notification bar or the Quickshell AI agents monitoring panel. Users have no quick visual indicator of their usage trends, model consumption, or session history, and cannot launch or manage `agy` natively through the system setup menu or default agent keybindings.

## Solution

Provide full first-class support for the Antigravity agent CLI (`agy`) within the Omarchy desktop ecosystem. Users can select Antigravity as their default coding agent via system menus or CLI. The desktop shell automatically tracks and visualizes active usage—displaying today's prompts, active sessions, weekly token trends, model distributions, and subscription tier in the notification bar and Quickshell agent dashboard. The integration handles unattended flag execution, passes interactive prompts for crash diagnosis and spontaneous queries, and discovers existing credentials without requiring custom user shims.

## User Stories

1. As a developer using Omarchy, I want to select Antigravity as my default coding agent from the Omarchy Setup menu, so that I can configure my preferred agent without editing config files by hand.
2. As a developer using Omarchy, I want to run `omarchy default agent agy` from the command line, so that I can switch my default agent programmatically in scripts or shell sessions.
3. As a developer using Omarchy, I want aliases like `antigravity` and `antigravity-cli` to resolve to the same agent, so that I don't have to guess the exact abbreviated identifier.
4. As a developer using Omarchy, I want Omarchy to install `agy` via `mise` automatically if it is not already installed when chosen as the default agent, so that setup is frictionless.
5. As a developer using Omarchy, I want pressing the default agent keyboard shortcut to launch `agy` in a dedicated terminal window with unattended permissions enabled (`--dangerously-skip-permissions`), so that I can begin working immediately without redundant confirmation prompts.
6. As a developer using Omarchy, I want running `omarchy agent prompt "..."` to launch `agy` with the prompt pre-seeded interactively, so that I can ask quick questions that transition smoothly into interactive pairing sessions.
7. As a developer using Omarchy, I want system crash notifications to invoke `omarchy agent crash <pid>` and seed the coredump diagnosis into `agy`, so that I can immediately debug application failures using my preferred agent.
8. As a developer using Omarchy, I want the AI agents widget in the top bar to show an icon when Antigravity usage is detected, so that I have immediate access to my AI activity metrics.
9. As a developer using Omarchy, I want the bar widget to self-hide if no agent usage is recorded, so that my panel stays clutter-free when AI agents are not in use.
10. As a developer using Omarchy, I want left-clicking the bar icon to open the Quickshell agent panel featuring Antigravity, so that I can review my daily and weekly metrics.
11. As a developer using Omarchy, I want right-clicking the bar icon to launch my default agent terminal directly, so that I have a fast mouse shortcut to start coding.
12. As a developer using Omarchy, I want middle-clicking the bar icon to cycle through my enabled AI subscriptions, so that I can check multiple providers quickly.
13. As a developer using Omarchy, I want to see the Antigravity SVG mark and brand name in the panel hero section, so that the active agent is clearly recognizable.
14. As a developer using Omarchy, I want to see my subscription tier badge (e.g., "Gemini Code Assist") in the hero header, so that I know which service tier is active.
15. As a developer using Omarchy, I want to see today's prompt count and session count on the current day's entry, so that I understand my daily volume of interactions.
16. As a developer using Omarchy, I want to see a 7-day token usage bar chart in the panel, so that I can monitor my weekly usage trends at a glance.
17. As a developer using Omarchy, I want the bar chart to highlight today's row and scale the bars relative to the highest token day of the week, so that proportions are intuitive.
18. As a developer using Omarchy, I want to see token usage broken down by model, so that I know how much compute is consumed across different model tiers (e.g. Flash vs Opus).
19. As a developer using Omarchy, I want to hover over a model row to inspect its input, output, and cache-read token counts, so that I can analyze prompt caching efficiency.
20. As a developer using Omarchy, I want the collector to authenticate seamlessly against the desktop secret service without prompting for credentials, so that usage updates run silently in the background.
21. As a developer using Omarchy, I want the panel to display "Waiting for auth" with a helpful tip if credentials are missing or expired, so that I know how to re-authenticate.
22. As a developer using Omarchy, I want usage scans to be cached with incremental file modification checks, so that opening the dashboard is instant and causes no disk thrashing.
23. As a developer using Omarchy, I want running `omarchy agent usage-update` or pressing 'r' inside the panel to perform a fresh update, so that I can see the latest turn data immediately.
24. As a developer using Omarchy, I want the scan to cover the last 30 days of conversations, so that performance remains consistent over months of heavy agent usage.
25. As a developer using Omarchy, I want my cross-device sync configuration to merge Antigravity tokens across machines if sync mode is enabled, so that my multi-machine statistics are unified.
26. As a developer using Omarchy, I want to see live five-hour and weekly rate limits and refresh countdowns in the panel for both Gemini and third-party models, so that I can anticipate when capacity will replenish without manually running CLI commands.

## Implementation Decisions

### Architectural Seams & Omarchy Agent Model
- Native integration follows the standard Omarchy agent provider pattern: an agent definition in the default selector, execution dispatch in the launcher, a menu registration, and an independent usage collector script.
- The primary agent identifier is established as `agy`, with backward-compatible aliases for `antigravity`, `antigravity-cli`, `gemini`, and `gemini-cli`.
- Display name is "Antigravity".

### Execution Dispatch & Toolchain
- Unattended terminal execution invokes the CLI with `--dangerously-skip-permissions`.
- Interactive prompt invocation passes the prompt through `--prompt-interactive`.
- Default packaging binds to the `antigravity-cli` tool in the `mise` registry, providing automated provisioning during setup.

### Data Collection & Usage Pipeline
- An independent collector script `omarchy-agent-usage-agy` runs under the control of the central usage update runner.
- The collector prints a single display-ready JSON record conforming to the Omarchy agent usage schema:
  - Protocol: `schemaVersion: 1`
  - Identification: `id: "agy"`, `name: "Antigravity"`
  - Status & Tier: `ready: true`, `tierLabel: "<tier-name>"`, `usageStatusText`, `authHelpText`
  - Metrics: `todayPrompts`, `todaySessions`, `todayTotalTokens`, `todayTokensByModel`
  - Historical Activity: `recentDays` (7-day daily token counts), `activeDays`, `activeDates`
  - Granular Model Breakdown: `modelUsage` mapping model IDs to input, output, cache-read, and cache-write counters
  - Rate Limits: `limits` array containing primary `Session (5-hour)` and `Weekly (7-day)` meters alongside attached `Claude/GPT` scoped meters
- Token and prompt metrics are extracted from local transcript files and history records for sessions touched within a 30-day window.
- Model names are resolved from local conversation metadata and normalized for human-friendly display in the UI.
- Authentication tokens are extracted from the desktop secret service (`service=gemini`, `username=antigravity`) to query Google's Cloud Code Assist APIs.
- Client Identification & Header Requirements: Google's Cloud Code Assist API gateway enforces that callers identify with an Antigravity user agent (`User-Agent: antigravity/<version>`). Calls omitting this header are rejected with error `#3501 (SUBSCRIPTION_REQUIRED)`. The collector dynamically queries the client version from `agy --version` and caches it in memory, falling back to a default version (`1.3.5`) if the binary is absent or unreachable.
- Quota & Rate Limit Architecture:
  - Account plan and tier metadata are resolved via `POST https://cloudcode-pa.googleapis.com/v1internal:loadCodeAssist`. When `paidTier` is present (such as `g1-pro-tier` for Google One AI Premium accounts), it takes precedence over fallback `currentTier` (`free-tier`) to display `"Google AI Pro"`.
  - Detailed quota buckets and replenishment timestamps are fetched via `POST https://cloudcode-pa.googleapis.com/v1internal:retrieveUserQuotaSummary`.
  - The endpoint returns quota grouped by model family (`Gemini Models` and `Claude and GPT models`), each with a 5-hour rolling window (`5h`) and a weekly window (`weekly`), providing `remainingFraction` and `resetTime`.
  - The collector normalizes `remainingFraction` into Omarchy's utilization metric: `percent = 1.0 - remainingFraction`.
  - Gemini limits map to native Omarchy windows (`Session (5-hour)` with title `Session`, and `Weekly (7-day)` with title `Weekly`). Third-party models map to scoped sub-meters (`Claude/GPT (5-hour)` with title `Claude/GPT Session`, and `Claude/GPT (7-day)` with title `Claude/GPT Weekly`), which Quickshell attaches to the primary meter rows.
  - If `retrieveUserQuotaSummary` returns HTTP 403 or context errors (e.g. unmetered accounts), the collector catches the error and degrades gracefully without failing (`limits: []`), matching `agy`'s internal `doRefreshQuota` behavior.
- Local State vs Remote Quota Findings: Quota percentages, limits, and replenishment timestamps are managed strictly in-memory by `agy` and are never serialized to disk or local caches under `~/.gemini/antigravity-cli/`. Active sessions are marked locally via exclusive `flock` locks on 0-byte files under `~/.gemini/antigravity-cli/presence/<conversation_id>.lock`. Direct calls to Google's Cloud Code Assist API are the authoritative source for quota metrics.
- Cached scan results are stored in the user cache directory with file timestamp validation to ensure fast repeated panel openings.

### Desktop Shell & UI Branding
- High-resolution SVG brand marks (`agy.svg` and `agy-light.svg`) are bundled in the agents shell plugin asset directory using the official Antigravity wave mark:
  ```xml
  <svg viewBox="0 -960 960 960" width="22" height="22" fill="currentColor" xmlns="http://www.w3.org/2000/svg">
    <path d="M482.04-859.61Q557.08-860 597-803.84t70,127.77q25.23,61.92 49.23,137.5t50.93,151.03q26.3,72.47 57.65,137.05T897-139.62q6.46,8.08 6.46,16.96T897-107.69t-16.35,6.38t-17.1-6.11q-76.71-64.2-122.32-140.01T646.48-376.6q-34.64-38.4-74.56-61.21t-89.63-22.42q-49.72-0.38-89.89,22.42T317.82-376.6q-49.36,53.37-94.98,129.18T100.52-107.42q-7.21,6.42-16.91,6.11t-16.54-6.58q-6.46-5.88-6.27-14.77t6.65-16.96q40.46-45.92 71.81-110.69t57.65-137.24Q223.85-463 247.85-538.3t49.23-137.77q29.18-71.36 69.55-127.64t115.41-55.9Zm-87.46,93.88q-31.27,48.5-53.88,102.27q-35.31,81.54-60.92,169.19T222.31-326.61q24.73-33.65 48.21-62.32t48.87-51.52q36.15-31.54 75.31-48.35t87.35-16.81t87.35,16.81t75.69,48.35q25,22.85 48.48,51.52t48.21,62.32q-31.85-80-57.46-167.65T623.77-663.46q-23-53.77-54.27-102.27t-87.46-48.5t-87.46,48.5Z"/>
  </svg>
  ```
- The Setup > Default Agent menu entry provides clean Antigravity branding positioned in alphabetical order.

## Testing Decisions

### Good Test Principles
- Tests must execute the external boundaries (CLI executables, exit codes, output JSON schema, environment variables) rather than calling internal helper functions.
- All tests must run hermetically using temporary directories for `$HOME`, `$XDG_STATE_HOME`, and `$XDG_CACHE_HOME` without mutating user configurations.

### Modules Tested
1. **Collector Executable (`test/shell.d/agent-usage-agy-test.sh`)**:
   - Tests unauthenticated state: verifies the collector outputs `ready: false` with appropriate help text when no secret is stored.
   - Tests authenticated state: verifies live tier resolution (`tierLabel`) and proper JSON schema formatting with stubbed API responses.
   - Tests rate limits parsing: verifies that `retrieveUserQuotaSummary` payloads containing model groups and buckets calculate utilization percentage (`1.0 - remainingFraction`) and format `Session (5-hour)`, `Weekly (7-day)`, and scoped `Claude/GPT` windows with normalized ISO reset timestamps.
   - Tests rate limits fallback: verifies that when `retrieveUserQuotaSummary` returns HTTP 403 or fails, the collector degrades gracefully to `loadCodeAssist` limits or an empty array with `ready: true`.
   - Tests transcript scanner: verifies correct calculation of daily tokens, prompt counts, and model splits from synthetic transcript and history files.
   - Tests caching behavior: verifies that cached scans are reused until invalidated.
2. **Menu & Agent Defaults (`test/shell.d/menu-test.sh`)**:
   - Asserts that `setup.default.agent.agy` is present in the menu model with correct icon, label, and alphabetical ordering.
3. **Usage Update Runner Integration (`test/shell.d/agent-usage-update-test.sh`)**:
   - Verifies that `omarchy-agent-usage-update` discovers and runs the `agy` collector in parallel and writes to the state directory.

### Prior Art
- Modeled directly on `test/shell.d/agent-usage-grok-test.sh`, `agent-usage-claude-limits-test.sh`, and `test/shell.d/menu-test.sh`.

## Out of Scope

- Reverse-engineering internal proprietary quota algorithms beyond what the Code Assist service exposes.
- Modifying internal storage engines or databases created by the `agy` binary.
- Managing background daemons or cloud workstations unrelated to local desktop coding.
- Supporting legacy deprecated Gemini CLI versions that do not support the Antigravity command line interface.

## Further Notes

- In the current upstream codebase (`quattro` branch of `basecamp/omarchy`), initial hooks for launching `agy` and the menu item have already been scaffolded, but the usage collector, test suite, and SVG assets were completely missing. This specification directly addresses that gap.
- Since an external issue tracker is not currently configured in the environment, this spec is saved in the repository at `plans/antigravity-support.md`.
