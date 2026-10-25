# Martin's OpenRig setup

What I know about Martin's two-machine setup, what the OpenRig repo says about each part, and
what I'd need to know to make it reproducible. Written from the project instructions, not from
looking at the machines, so everything about the live machines is **[unverified]**.
Companion to [OPENRIG-MAP.md](OPENRIG-MAP.md). Checked against fork `main` at `953ce6d`.

> **Privacy note.** This fork is public on GitHub. I left out the Tailscale IPs and Linux
> usernames on purpose. They are in the project instructions; add them here only if you're happy
> for them to be public, or keep them in a local, untracked file.

## The machines

| | Work PC | Home PC |
| --- | --- | --- |
| Host name | `work-wsl` | `home-wsl` |
| OS | Windows + WSL2 Ubuntu | Windows + WSL2 Ubuntu (distro disk on `D:`) |
| Tailscale address | `<work-wsl tailnet IP>` | `<home-wsl tailnet IP>` |
| Linux user | `<work user>` | `<home user>` |
| OpenRig | 0.7.0, own daemon + own kernel | 0.7.0, own daemon + own kernel |
| Node | 22 via nvm | 22 via nvm |
| Terminal provider | herdr 0.9.5 | herdr 0.9.5, with `work` saved as a herdr machine |
| Network | Tailscale inside WSL, `--ssh`, `--accept-dns=false` | same |
| Autostart | something starts things at boot (details unknown) | same |

## How each layer fits OpenRig

### WSL2 itself

- OpenRig supports Windows only through WSL2, but **its automated tests don't run on WSL2**. The
  one reported working setup (OpenRig 0.6.6, Ubuntu 24.04, Node 24, tmux 3.4) did **not** test the
  kernel, herdr, reboot, suspend/resume, cross-host or long-running work: exactly the things this
  setup relies on. Source: `docs/reference/getting-started.md` ("WSL2: a reported working setup").
- What mattered there: install OpenRig and its state on the Linux filesystem, and use Linux-native
  Node and harnesses, never the Windows `npm`/`node`/`claude` that WSL can inherit on `PATH`.
  Check with: `which -a node npm rig claude tmux`
- 0.6.10 fixed a WSL2-specific false "port in use" on first daemon start
  (`packages/cli/src/daemon-lifecycle.ts`, comment "WSL2 can silently drop probes").

### tmux

- Ubuntu 22.04's stock tmux 3.2a can crash the whole tmux server when OpenRig checks a pane that
  just closed, taking every agent with it. Use tmux 3.4 or later. Ubuntu 24.04 ships 3.4.
  Source: `CHANGELOG.md` 0.7.0 "Known issues", `docs/reference/getting-started.md`.
  Check with: `lsb_release -ds; tmux -V`
- Since 0.7.0, new seats no longer inherit `tmux set-environment -g` values. Put keys in a
  seat's `env` or `rig up --env`.

### Node 22 via nvm

- Supported (`package.json` engines `^22 || ^24`). nvm keeps a separate global package set per
  Node version, so `rig` must be installed under the same Node your boot script uses.
- Risk: a boot script runs a non-interactive shell where `~/.nvm/nvm.sh` isn't loaded, so `node`
  and `rig` aren't on `PATH`. A Mac report had Claude hooks failing with `node: command not found`
  for this reason (`docs/reference/getting-started.md`). Since 0.6.13 generated Claude hook
  commands call the daemon's Node by absolute path, which helps once the daemon itself started
  under the right Node.

### The daemon, the kernel and "autostart"

- **OpenRig ships no boot service.** There is no systemd unit, Windows task or launchd job in the
  repo; `packages/cli/src/daemon-auto-start.ts` only means a `rig` command can start a stopped
  daemon. Whatever starts your daemon at boot is your own script.
- When the daemon starts and finds a kernel whose seats were all lost (as after a reboot), it
  restores that kernel by itself. A kernel stopped with `rig down kernel` stays down. Other rigs
  are not restored automatically: run `rig start` (or `rig up <rig> --existing`).
  Source: `docs/reference/getting-started.md` ("Start the kernel and check its state").
- After boot, check rather than assume: `rig status && rig ps --nodes --rig kernel`
- WSL2 facts that matter here (general WSL behaviour, not from this repo, **[unverified]** for
  your machines): a WSL distro only starts when something launches it after Windows logs in;
  without a Windows-side trigger nothing in WSL runs at boot. Systemd inside WSL is opt-in
  (`/etc/wsl.conf`, `[boot] systemd=true`). WSL can shut an idle VM down, which would kill tmux
  and every agent.

### Tailscale inside WSL, and the two daemons talking

- Each PC runs its own daemon and kernel, which matches OpenRig's model: every machine is its own
  instance, and paired instances form a fleet (`docs/reference/topology-naming.md`).
- By default the daemon listens on `127.0.0.1` only. For the other PC to reach it, start it on
  its Tailscale address. The code allows a Tailscale (100.64.0.0/10) bind without a token
  (`packages/daemon/src/middleware/auth-bearer-token.ts`). Single-line form:
  `rig daemon start --host <this-pc-tailnet-IP> --port 7433`
- Pair each direction once (one approval on the target each time). From work:
  `rig host pair http://<home-wsl tailnet IP>:7433 --id home` and from home:
  `rig host pair http://<work-wsl tailnet IP>:7433 --id work`
- Then check: `rig host list && rig host doctor home && rig ps --host home --nodes`
- `--accept-dns=false` means MagicDNS names won't resolve inside WSL, so pair by IP. The daemon
  accepts requests addressed to its IP, its hostname or its Tailscale name; any other alias needs
  `OPENRIG_ALLOWED_HOSTS` in the target daemon's environment (`docs/reference/browser-access.md`).
- Cross-machine OpenRig carries messages, screen captures and queue tasks only. It does not sync
  repos, files, logins or agent context; each PC needs its own checkout and harness sign-ins.
- `--ssh` (Tailscale SSH) is not needed for HTTP pairing. It matters for OpenRig hosts registered
  with `transport: ssh` (`rig host add --transport ssh ...`) and for herdr's remote machine.
  Terminal tiles for seats on an HTTP-registered host can't be composed ("tiles need ssh"), per
  `packages/daemon/specs/agents/shared/skills/core/openrig-herdr/SKILL.md`.

### herdr 0.9.5

- herdr is OpenRig's default terminal provider; OpenRig talks to its control socket
  (`~/.config/herdr/herdr.sock`). The protocol in `packages/daemon/src/domain/terminal/herdr-transport.ts`
  was captured against herdr 0.7.1. Nothing in the repo names 0.9.5 as tested **[unverified]**.
- herdr's own "saved machine" (`work` on the home PC) is a herdr feature. OpenRig doesn't know
  about it: tiles you see through it are the work PC's daemon's sessions, viewed over SSH.
- 0.7.0 known issue: the TUI's terminal button refuses inside a Herdr pane reached over SSH.
  Workaround from the release notes, single line:
  `env -u SSH_CONNECTION -u SSH_CLIENT -u SSH_TTY rig tui`
  0.7.1 says it fixes this and that tiles come back after a herdr restart.

### Claude-only until 30 Oct

- On a **fresh** instance, `rig daemon start` picks the kernel variant from which logins pass:
  Claude-only only when `codex login status` *fails* (`packages/daemon/specs/rigs/launch/kernel/rig-claude-only.yaml`).
  If Codex is still signed in but out of quota, it would pick the mixed kernel. An existing kernel
  isn't re-chosen on later starts.
- Shipped teams that use Codex: `starter` (reviewer), `factory` (lead, QA, one reviewer),
  `workshop` (lead, reviewer). Ask the operator to adapt `starter`, or install workshop with
  `--preset all-claude`. My own RigSpecs in `my-rigs/` should use `runtime: claude-code` only.

## Open questions (to make this reproducible)

1. Ubuntu version and `tmux -V` on each PC? (3.2a is a known crash.)
2. What exactly autostarts, and how? A Windows Task Scheduler entry running `wsl.exe ...`, a WSL
   `[boot] command`, systemd units, or a shell profile? Please paste the script(s).
3. Is systemd enabled in each distro (`/etc/wsl.conf`)? Is tailscaled run by systemd or by hand?
4. Does the boot script wait for Tailscale before starting the daemon? A bind to the 100.x address
   fails if `tailscale0` isn't up yet.
5. Which `--host` does each daemon bind today (`rig daemon status`), and have the PCs been paired
   (`rig host list`)? If not paired, how do the two instances interact today besides herdr?
6. How does the boot script load nvm (absolute path to Node 22, or sourcing `nvm.sh`)? Exact Node
   patch version?
7. Do you keep the WSL VM alive (`.wslconfig` `vmIdleTimeout`, or something holding a session)?
   Do the PCs sleep or hibernate?
8. Where do project repos live in each distro (Linux home vs `/mnt/c` or `/mnt/d`)? The WSL2 note
   says keep everything on the Linux filesystem.
9. Is `OPENRIG_HOME` the default `~/.openrig` on both? Any non-default `rig config` values
   (`rig config get` output) or `daemon.host` in config?
10. Which Claude Code and Codex versions, and how are they installed (npm global under nvm, native
    installer)? Is Codex signed in right now on either PC?
11. herdr: how was 0.9.5 installed (`rig setup` or herdr's installer), and how is the `work`
    machine defined (SSH target over Tailscale SSH)? Does the work PC also save `home`?
12. Do you want the kernel to come back after reboot only (the default), or your project rigs too
    (`rig start` in the boot script)?
13. Upgrade plan: npm `latest` is now 0.7.1 (25 Oct). Stay on 0.7.0 for now, or upgrade both PCs
    together? Mixed versions across paired hosts are not covered by the docs.
14. OK to record the Tailscale IPs and usernames in this public fork, or keep them local?
