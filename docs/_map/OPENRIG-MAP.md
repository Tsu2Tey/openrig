# OpenRig map

A wide-angle map of this repository for agents and for Martin. It points at the files that are
the source of truth; it does not replace them. Where this page and the code disagree, the code
wins (the same rule `ARCHITECTURE.md` and `.claude/skills/developing-openrig/SKILL.md` state).

- **Checked against:** fork `main` at `953ce6d` (24 Oct 2026). That commit is *after* the 0.7.0
  release and *before* the 0.7.1 release: npm `latest` is `0.7.1` (published 25 Oct 2026) and
  openrig.dev already documents 0.7.1, but this fork's `CHANGELOG.md` stops at 0.7.0. Martin's
  machines run 0.7.0. So some commands named in this checkout's docs are newer than what is
  installed. Always check `rig --version` and `rig <command> --help` on the machine.
- **Markers:** **[unverified]** means I read it in docs but did not trace it in code, or it depends
  on Martin's machines, which I can't see.
- **Deeper maps, in order of trust:** the code; `rig <cmd> --help`; `docs/reference/` (ships with
  the package); `ARCHITECTURE.md`; `docs/as-built/codemap.md` and `docs/as-built/architecture/*`
  (each carries a `last-verified-against-source` commit); the capability map
  `packages/daemon/context-packs-src/onboarding-width/public-what-you-can-do.md`.

## The one-paragraph picture

OpenRig runs a team of ordinary Claude Code / Codex / Pi sessions inside tmux and coordinates
them. A local **daemon** (Hono HTTP on port 7433, SQLite in `~/.openrig`) owns all state. People
and agents drive it through the **`rig` CLI**, the **TUI** and an **MCP server**, all thin HTTP
clients. A **RigSpec** (`rig.yaml`) declares a **rig** made of **pods** of **seats**; each seat
has a stable address `pod-member@rig`. The daemon boots its own small team, the **kernel**,
whose operator helps you launch your project teams. Messages are typed into terminals; durable
work lives in the **queue**; plans live on disk as **missions and slices**.

---

## 1. Daemon

The background service that holds everything: SQLite state, tmux sessions, runtime adapters,
delivery, sweeps and the HTTP API. It is the only writer of truth; every other surface asks it.
It listens on `127.0.0.1:7433` by default; it can bind a Tailscale or other private address, and
refuses a public/LAN bind without a bearer token. State lives in `$OPENRIG_HOME` (default
`~/.openrig`), database `openrig.sqlite`. Restarting it does not stop agents' tmux sessions.

- **Commands:** `rig daemon start [--host <ip>] [--port] [--no-kernel]`, `rig daemon stop`,
  `rig daemon status`, `rig daemon logs`, `rig status`, `rig doctor --json`, `rig config get|set|reset`.
- **Source of truth:** `packages/daemon/src/index.ts` (process entry), `packages/daemon/src/startup.ts`
  (`createDaemon()`), `packages/daemon/src/server.ts` (`createApp()`, route mounts),
  `packages/daemon/src/routes/`, `packages/daemon/src/domain/`, `packages/daemon/src/db/all-migrations.ts`,
  `packages/daemon/src/middleware/auth-bearer-token.ts` (loopback/Tailscale bind rules),
  `packages/cli/src/daemon-lifecycle.ts` (start/stop, dev-checkout vs bundled daemon),
  `packages/cli/src/commands/daemon.ts`, `docs/reference/instance-layout.md`.
- **Not provided:** a boot-time service. Nothing in the repo installs a systemd unit, launchd job
  or Windows task; "auto-start" in the code (`packages/cli/src/daemon-auto-start.ts`) means a CLI
  command starting a stopped daemon on demand.

## 2. CLI, TUI and MCP

`rig` (npm `@openrig/cli`, the only published package) is for both humans and agents; ~90
top-level commands, most offering `--json`. Bare `rig` in a terminal opens the TUI. The TUI is
the dashboard (topology, seats, Projects, Feed, Terminals, System). `rig mcp serve` exposes tools
such as `rig_up`, `rig_ps`, `rig_send` over stdio. The React web UI still ships but is in
maintenance mode.

- **Commands:** `rig --help` (read the whole list before deciding a command is missing),
  `rig tui`, `rig tui commands`, `rig mcp serve`, `rig whoami --json`.
- **Source of truth:** `packages/cli/src/index.ts` (`createProgram()`, every `program.addCommand`),
  `packages/cli/src/commands/`, `packages/cli/src/client.ts` (`DaemonClient`),
  `packages/cli/src/front-door.ts`, `packages/cli/src/mcp-server.ts`, `packages/cli/src/commands/mcp.ts`,
  `packages/tui/src/main.ts`, `packages/tui/src/daemon-client.ts`, `docs/as-built/cli-reference.md`.

## 3. Kernel

OpenRig's own small rig, named `kernel`, booted by `rig daemon start` from whichever providers
are signed in (Claude-only, Codex-only or mixed variant). Seats: `operator.agent` (the agent you
talk to first; it recommends and launches teams), `operator.human` (a terminal seat running
`rig tui`), `advisor.lead` (thinking partner), `queue.worker`. Kernel seats get a broader
"operational" permission default than project teams. After a reboot the daemon restores a
kernel whose seats were all lost; a kernel you stopped with `rig down kernel` stays down.

- **Commands:** `rig status` (the `Kernel:` line), `rig ps --nodes --rig kernel`,
  `rig terminal open saved:kernel --window`, `rig up kernel --existing [--fresh <seat...>]`.
- **Source of truth:** `packages/daemon/specs/rigs/launch/kernel/rig.yaml`,
  `.../kernel/rig-claude-only.yaml`, `.../kernel/rig-codex-only.yaml`, `.../kernel/agents/`,
  `packages/daemon/src/domain/kernel-boot.ts`, `packages/daemon/src/domain/kernel-boot-tracker.ts`,
  `packages/daemon/src/adapters/kernel-authority.ts` (kernel and team permission defaults),
  `packages/daemon/src/routes/kernel-status.ts`; docs: `docs/reference/getting-started.md`
  ("Start the kernel and check its state").

## 4. Seats, pods and rigs (RigSpec, AgentSpec, runtimes)

A **rig** is one team; a **pod** groups seats around a domain; a **seat** (member) is a role
with a stable address `{pod}-{member}@{rig}` (logical ID `pod.member`). A **RigSpec**
(`rig.yaml`, version `"0.2"`) declares pods, members (`agent_ref`, `runtime`, `profile`, `cwd`,
optional `model`, `effort`, `env`, `permission_policy`, `restore_policy`...), edges and a culture
file. An **AgentSpec** (`agent.yaml`) is the reusable blueprint a member points at. Runtimes:
`claude-code`, `codex`, `pi`, `omp`, `terminal`, `stub`. Edges: only `delegates_to` and
`spawned_by` change behaviour (launch order; `delegates_to` also routes queue escalation).

- **Commands:** `rig spec validate <rig.yaml>` → `rig spec preflight <rig.yaml>` → `rig spec audit <rig.yaml>`,
  `rig agent validate <dir>`, `rig specs ls|preview|add`, `rig up <spec|name> [--cwd] [--plan] [--name]`,
  `rig down <rig>`, `rig ps --nodes [--rig]`, `rig launch`, `rig grow`, `rig add`, `rig remove`,
  `rig shrink`, `rig edge add|remove`, `rig seat ...` (status, continue, set-cwd, set-permissions,
  set-effort), `rig export <rig>`, `rig create`.
- **Source of truth:** `packages/daemon/src/domain/rigspec-schema.ts`, `.../rigspec-preflight.ts`
  (`SUPPORTED_RUNTIMES`), `.../rigspec-instantiator.ts`, `.../startup-orchestrator.ts`,
  `.../node-launcher.ts`, `.../runtime-adapter.ts`, `packages/daemon/src/adapters/*-adapter.ts`,
  AgentSpec: `packages/daemon/src/domain/agent-manifest.ts`, `.../agent-resolver.ts`,
  `.../profile-resolver.ts`. Docs: `docs/reference/rig-spec.md`, `docs/reference/agent-spec.md`,
  `docs/reference/edge-types.md`, `docs/reference/topology-naming.md`.
- **Shipped teams:** `packages/daemon/specs/rigs/launch/` (`starter`, `factory`, `kernel`,
  `secrets-manager`; `first-project` is starter's alias), `packages/daemon/specs/rigs/focused/`
  (`code-review`, `research`, `pm`), agents in `packages/daemon/specs/agents/`. `workshop` is
  **not** built in: it is a bundle in `mvschwarz/openrig-world` (see Bundles). As shipped,
  `starter` has a Codex reviewer and `factory` puts the lead, QA and one reviewer on Codex.

## 5. Messaging

"The terminal is the wire": `rig send` types a signed message into another seat's prompt (and
refuses to type over a waiting permission prompt). `--verify` proves the text was staged, not
that it was read. Also: broadcast to many seats, per-rig chatrooms, screen capture, seat-scoped
transcripts, keyword search across what was said, and the stream (observations for the next
agent). Slack and "registered humans" are the gateway, a separate subsystem.

- **Commands:** `rig send <seat> "<msg>" [--verify] [--host]`, `rig broadcast`, `rig capture <seat>|--rig`,
  `rig chatroom send|wait|history`, `rig transcript <seat>`, `rig ask`, `rig stream emit|list|read`,
  `rig seat set-typing-guard`, `rig gateway human ...`, `rig slack ...`.
- **Source of truth:** `packages/daemon/src/domain/session-transport.ts`,
  `.../seat-delivery-guard.ts`, `.../chat-repository.ts`, `.../transcript-store.ts`,
  `.../stream-store.ts`, `packages/daemon/src/routes/transport.ts`, `.../routes/chat.ts`,
  `packages/daemon/src/domain/gateway/` (Slack, humans, quiet hours, modes),
  `packages/cli/src/commands/send.ts`, `.../chatroom.ts`, `.../ask.ts`;
  docs: `docs/reference/seat-delivery.md`, `docs/as-built/architecture/transport-and-transcripts.md`.

## 6. Queue (durable work) and watchdogs

A queue row is a task with one owner, a body and a transition history; it survives compaction,
handover and restarts. Messages are not tasks: a seat must record work with `rig queue create`.
Rows can be claimed, handed off atomically, blocked on something with a timed wake, or resolved
by a human. The daemon runs a wake ladder and stuck-row sweeps that escalate along
`delegates_to`. Watchdogs arm wakes in advance, because an agent can't wake itself.

- **Commands:** `rig queue create|show|list|claim|update|handoff|block|resolve|overdue|undelivered`,
  `rig parked`, `rig watchdog register|list|show|stop`.
- **Source of truth:** `packages/daemon/src/domain/queue-repository.ts`, `.../queue-wake-ladder.ts`,
  `.../queue-stuck-sweep.ts`, `.../queue-owner.ts`, `.../watchdog-scheduler.ts`,
  `packages/daemon/src/routes/queue.ts`, `packages/cli/src/commands/queue.ts`,
  `packages/cli/src/commands/watchdog.ts`; docs: `docs/as-built/architecture/coordination-primitive.md`.

## 7. Workflows

A workflow spec names steps and owners; each step becomes a queue packet, the next opens when it
closes, and failures or overdue steps route to the orchestrator. A project's required release
steps (`project.yaml`) plus a mission's own steps compile into one graph. Agents judge outcomes;
workflow completion is never acceptance by itself.

- **Commands:** `rig workflow specs|validate|compile|instantiate|run|status|show|trace|revise|route|resume|continue`
  (`continue` is read-only).
- **Source of truth:** `packages/daemon/src/domain/workflow-runtime.ts`, `.../workflow-projector.ts`,
  `.../workflow-validator.ts`, `packages/daemon/src/domain/workflow/`, `packages/daemon/src/domain/policies/`,
  `packages/daemon/src/routes/workflow.ts`, `packages/cli/src/commands/workflow.ts`;
  docs: `docs/as-built/architecture/workflow-runtime.md`, `docs/reference/project-release-profile.yaml`.

## 8. Scope, missions, slices and proof (the work tree)

Work lives on disk under the workspace (`$OPENRIG_HOME/workspace` by default): a project
(`SPEC.md`, `project.yaml`), missions, and slices with frontmatter and stable dot-IDs. The TUI's
Projects view reads it. Proof is separate: evidence is attached with `rig proof add`, and work
counts as done when a judge named in the proof policy accepts it.

- **Commands:** `rig scope mission ls|show|create`, `rig scope slice ls|show|create|progress|approve|close|ship`,
  `rig scope audit`, `rig proof add|judge|show`, `rig workspace doctor|validate`, `rig project ...`.
- **Source of truth:** `packages/daemon/src/domain/scope/`, `packages/daemon/src/domain/slices/`,
  `packages/daemon/src/domain/proof/`, `packages/daemon/src/domain/workspace/`,
  `packages/daemon/src/routes/missions.ts`, `.../routes/slices.ts`, `.../routes/scopes.ts`,
  `packages/cli/src/commands/scope.ts`, `.../proof.ts`; docs: `docs/reference/project-workspace.md`,
  `docs/reference/sdlc-conventions.md`, `docs/as-built/architecture/workspace-primitive.md`.

## 9. Context library and skills

What a seat knows when it arrives. The **context library** (`$OPENRIG_HOME/context`, setting
`context.root`) serves packs by address; every shipped skill and static pack becomes a context
pack at package time, and `docs/reference/` is served too (`rig context get help`). **Skills**
have one editable "home" each and generated copies elsewhere (`npm run mirror-skills`). Seats get
the skills their AgentSpec profile selects; the person-facing `rigs` skill is installed into
`~/.claude/skills` and `~/.agents/skills` by the daemon.

- **Commands:** `rig context list|get <ref>|show|sync|add --git|profile|work-install|recap-write`,
  `rig skill ...`, `rig plugin show|used-by|validate`.
- **Source of truth:** `packages/daemon/src/domain/context-packs/`, `packages/daemon/context-packs-src/`
  (hand-written packs), `scripts/generate-context-packs.mjs`, `packages/daemon/src/routes/context-packs.ts`,
  `packages/cli/src/commands/context.ts`, `packages/cli/src/lib/work-install.ts`.
  Skills: `packages/daemon/specs/agents/shared/skills/` (home for spec skills),
  `packages/daemon/assets/plugins/openrig-core/skills/` (plugin-only skills), `skills/_canonical/`
  (generated public mirror, do not edit), `skills/rigs/` (person-facing skill),
  `scripts/skill-edge-layout.generated.json`; index: `skills/_canonical/core/openrig-skills/SKILL.md`.

## 10. Continuity: snapshots, restore, handover, compaction

`rig down --snapshot` captures a rig; `rig up <name>` / `rig up <name> --existing` restores it,
resuming each conversation where possible and reporting per seat (resumed, fresh-primed,
awaiting-decision, attention_required, failed). `rig start` brings everything back after a
reboot. A full seat can be replaced in place by `rig handover` with a packet; Claude seats
can opt in to managed compaction (off by default). `continuity_policy` in a RigSpec is stored but
**nothing acts on it yet** (`docs/reference/rig-spec.md`).

- **Commands:** `rig down <rig> --snapshot`, `rig up <rig> --existing`, `rig start [--last]`,
  `rig restore-check`, `rig snapshot ...`, `rig restore ...`, `rig handover <seat> --dry-run`,
  `rig compact`, `rig fork`, `rig agent-image ...`.
- **Source of truth:** `packages/daemon/src/domain/snapshot-capture.ts`, `.../restore-orchestrator.ts`,
  `.../restore-check-service.ts`, `.../periodic-snapshot-scheduler.ts`, `.../seat-handover-service.ts`,
  `.../claude-compaction-enforcer.ts`, `packages/daemon/src/domain/agent-images/`,
  `packages/daemon/src/routes/snapshots.ts`, `packages/cli/src/commands/snapshot.ts`,
  `.../restore.ts`, `.../seat.ts`; docs: `docs/as-built/architecture/lifecycle-snapshot-restore.md`,
  `docs/reference/getting-started.md` ("Incomplete setup and restart").

## 11. Hosts and multiple machines

Each machine runs its own daemon and rigs; registered hosts form a fleet. `rig host pair` does
one-address pairing with an approval on the target and stores a bearer token. `--host <id>` then
works on `send`, `capture`, `ps`, `whoami`, `queue create` and `queue handoff`. It does **not**
sync repos, files, logins or context. Use a trusted private network (Tailscale is what the
maintainer tests); never expose the daemon publicly. The target must bind a reachable address
(`rig daemon start --host <private-ip>`).

- **Commands:** `rig host pair http://<ip>:7433 --id <name>`, `rig host add|list|doctor|select|rename`,
  `rig file copy`, `rig ps --host <id> --nodes`, `rig send <seat> "..." --host <id>`.
- **Source of truth:** `packages/daemon/src/domain/hosts/` (registry reader/writer, learned
  bindings, remote HTTP), `packages/cli/src/commands/host.ts`, `packages/daemon/src/routes/hosts.ts`,
  `packages/daemon/src/middleware/auth-bearer-token.ts`; registry file `~/.openrig/hosts.yaml`,
  tokens `~/.openrig/secrets/host-<id>.token`; skill
  `packages/daemon/specs/agents/shared/skills/core/cross-host-rig-commands/SKILL.md`;
  docs: `docs/reference/browser-access.md`, openrig.dev `/guides/multiple-machines` (written for 0.6.6).
- **[unverified]** Cross-host work on WSL2 specifically. The repo's WSL2 notes list cross-host
  operation as not tested.

## 12. Terminal providers (herdr, cmux, Rex, plain tmux)

Agents always run in tmux sessions owned by the daemon. A terminal provider only *shows* them:
`rig terminal open <view>` composes a view (a rig, `pod:`, `mission:`, `slice:` or a saved view
from `$OPENRIG_HOME/terminal-views.yaml`) as tiles that are nested `tmux attach` clients.
**herdr** is the default provider (talked to over its control socket, at arm's length because of
its AGPL licence); `rig setup` installs it unless `--no-herdr`. cmux is supported; Rex was added
on `main` after 0.7.0. Without a provider, plain tmux is used. Closing a tile never stops an agent.

- **Commands:** `rig terminal open <view> [--provider herdr|cmux] [--window]` (`--provider rex` and `--session` are newer than 0.7.0),
  `rig terminal views`, `rig terminal status`, `rig tui --shared` (dashboard-only fallback).
- **Source of truth:** `packages/daemon/src/domain/terminal/` (`herdr-adapter.ts`,
  `herdr-transport.ts`, `herdr-sessions.ts`, `cmux-provider-adapter.ts`, `rex-adapter.ts`,
  `terminal-service.ts`, `terminal-views-store.ts`, `view-composer.ts`),
  `packages/daemon/src/adapters/tmux.ts`, `packages/cli/src/commands/terminal.ts`,
  `packages/cli/src/commands/setup.ts` (herdr install); skills `.../skills/core/openrig-herdr/SKILL.md`,
  `.../skills/core/openrig-cmux/SKILL.md`; docs: `docs/reference/terminal-sessions.md`.
- **[unverified]** herdr 0.9.5. The herdr socket protocol in the code was captured against herdr
  0.7.1 (comments in `herdr-transport.ts`); no newer herdr version is named. Tiles for a seat on an
  HTTP-registered host can't be composed ("tiles need ssh"); herdr's own saved remote machines are
  a herdr feature, separate from OpenRig's host registry.

## 13. Bundles

A **RigBundle** is a whole team someone else can install: `rig.yaml`, its agents (all `local:`
refs inside the folder), docs, optional `bundle.yaml` (preconditions, min CLI version) and
`configurations.yaml` (harness presets such as all-Claude). Share it as a GitHub folder link
pinned to a full commit; `rig bundle inspect` shows what it will run and write first. The public
listing is openrig.dev/rigs, fed by `mvschwarz/openrig-registry`; `workshop` lives in
`mvschwarz/openrig-world`.

- **Commands:** `rig bundle check <folder>`, `rig bundle create <rig.yaml> -o x.rigbundle`,
  `rig bundle inspect <file|link>`, `rig bundle install <file|link> --target <dir> [--plan]`,
  `rig bundle configurations <rig.yaml>`, `rig up <github-link> --target <dir> --cwd <project> [--preset all-claude]`.
- **Source of truth:** `packages/daemon/src/domain/pod-bundle-assembler.ts`, `packages/daemon/src/domain/bundle-*.ts`,
  `packages/daemon/src/domain/bootstrap-orchestrator.ts`, `packages/daemon/src/routes/bundles.ts`,
  `packages/daemon/src/routes/up.ts`, `packages/cli/src/commands/bundle.ts`,
  `packages/cli/src/lib/bundle-source.ts` (GitHub links), `packages/cli/src/lib/bundle-check.ts`;
  docs: `docs/reference/publishing-a-rig-bundle.md`, `docs/reference/rig-bundle.md`,
  `docs/reference/bundle-formats.md`, `docs/as-built/architecture/packaging-bootstrap-bundles.md`.

## Permissions (cross-cutting)

Not a system of its own, but it shapes every launch. Project-team seats with no policy get a
per-launch "team default" (Claude: ordinary `rig` commands, reads and tests allowed; lifecycle
commands like `rig up`/`rig down` still ask). Kernel seats get a broader operational default.
`permission_policy: builtin:locked|standard|open|yolo|auto` on a rig or member changes that;
`--non-interruptive` only hides first-launch warnings for full-bypass seats.
Source: `packages/daemon/src/domain/permission-policy/policy-ref.ts`,
`packages/daemon/src/domain/native-permission-store.ts`, `packages/daemon/src/adapters/kernel-authority.ts`,
`packages/daemon/src/adapters/non-interruptive.ts`; docs: `docs/reference/rig-spec.md`
("Team launch defaults"), `docs/reference/non-interruptive-mode.md`.

## Repo guards to know before committing here

- `scripts/check-docs-guard.mjs` (part of `npm test`) only allows tracked docs under
  `docs/as-built/`, `docs/reference/`, `docs/releases/` and `docs/DESIGN.md`; `.gitignore`
  ignores the rest of `docs/`. **This file is in `docs/_map/`, so it is force-added and makes that
  guard fail in this fork.**
- `docs/reference/` ships inside the npm package; don't put personal notes there.
- Never hand-edit `scripts/*.generated.json`, `skills/_canonical/` or `packages/daemon/context-packs/`.
- Before changing code, read `.claude/skills/developing-openrig/SKILL.md` and `docs/as-built/arteries.md`.
