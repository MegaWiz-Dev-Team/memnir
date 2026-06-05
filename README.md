# Memnir 🐦‍⬛

*memory + [Mímir](https://en.wikipedia.org/wiki/M%C3%ADmir)* — share [Claude Code](https://docs.claude.com/en/docs/claude-code) memory **across machines** and **across every session**, peer-to-peer over [Tailscale](https://tailscale.com). No cloud.

> 🇹🇭 [อ่านภาษาไทย](README.th.md)

Claude Code stores memory per-project under `~/.claude/projects/<encoded-path>/memory/`, tied to one machine and one working directory — open another machine or another project and none of it follows you. Memnir unifies it into **one pool** that every session on every machine reads and writes, and syncs only the memories you choose between machines.

## Architecture

```
every project, every machine ──symlink──► ~/.claude/memnir/   (canonical store)
                                                │
              SessionStart hook                 │            Stop hook
              memnir start ───────────────┴───────────────── memnir push (async)
              (autolink current project + sync)              (propagate new memory)
                                                │
                              rsync -auz over Tailscale
                       only `scope: shared` files · newest-wins · never deletes
```

1. **Store** — `~/.claude/memnir/`, the single place real memory files live on each machine.
2. **Symlink** — every project's `memory/` dir points here, so all sessions share one pool.
3. **Sync** — two-way `rsync` between machines over Tailscale, filtered by scope.

## Install

Memnir is a single **Rust binary** (pure std, no external crates). On each machine:

```bash
git clone https://github.com/MegaWiz-Dev-Team/memnir
cd memnir
./install.sh
```

`install.sh` builds with `cargo build --release`, installs the binary to `~/.local/bin/memnir`, adds a shell alias, symlinks all existing projects into the pool, installs the auto-sync hooks, and asks for the peer host.

> No `cargo` on a machine but same arch (Apple Silicon)? Build once and copy the binary: `scp target/release/memnir other-mac:.local/bin/`. `install.sh` falls back to a prebuilt binary if `cargo` is missing.

### Configure the peer

Each machine needs to know its peer (the *other* machine). Set it once — either an env var or a one-line config file:

```bash
echo 'you@other-mac' > ~/.claude/memnir.conf      # user@tailscale-host of the other machine
# or:  export MEMNIR_PEER=you@other-mac
```

No host data is baked into the binary.

## Usage

### Normally: nothing

After `./install.sh`, Memnir runs **automatically every session** via hooks: opening Claude pulls the latest memory and links the current project into the pool; when Claude writes a memory, it's pushed to the peer right after the turn.

### When you want to act

```bash
memnir share project_firestore_envs   # mark a memory shared (default is local) + push
memnir local debug_scratch_today      # un-share → back to local (no sync)
memnir list                           # which memories are shared vs local
memnir sync                           # manual two-way sync (hooks do this for you)
memnir doctor                         # health report: token footprint, issues, actions
memnir dash && open ~/.claude/memnir/dashboard.html   # knowledge-graph + token dashboard
memnir link                           # link the current project into the pool right now
```

### What `memnir doctor` shows

```
MEMNIR HEALTH ───────────────────────────────── your-laptop
inventory   231 memories   project:174 feedback:36 reference:21
scope       shared:195   local:36
tokens      index ~15.0k/session 🔴   pool ~195k
sync        peer you@other-mac ✓   drift: 0 files

⚠ ISSUES & ACTIONS
 🔴 index 15k always-on        → compact-index (Tier-0 split)
 🟠 21 broken [[links]]          → memnir fix-links
 🟡 66 isolated memories       → link them (graph: memnir dash)
```

### Command reference

| command | what it does |
|---|---|
| `memnir sync` | push + pull `scope: shared` only, then regenerate the index |
| `memnir push` / `pull` | one direction (shared only) |
| `memnir share <id>` | set a memory `scope: shared` and push it to the peer |
| `memnir local <id>` | remove the tag → local again (won't sync) |
| `memnir list` | list shared vs local memories |
| `memnir status` | store path, counts (shared:local), peer |
| `memnir start` | autolink current project + sync (run by the SessionStart hook) |
| `memnir link` | manually symlink the current project into the pool |
| `memnir doctor [--check]` | health report + actions (`--check` = quiet unless there's an issue; for hooks) |
| `memnir dash` | write `dashboard.html` (knowledge graph + token visualization) |

`<id>` is a memory name, with or without `.md`.

## Scope: shared vs local 🔑

Memnir syncs **only memories you intend to share across machines**, not everything. Controlled by a frontmatter field:

```yaml
---
name: project_firestore_envs
metadata:
  type: project
  scope: shared      # <- this line = sync across machines
---
```

- **`scope: shared`** → synced both ways over Tailscale.
- **no `scope`** (default) → **local**; stays on this machine only.
- `MEMORY.md` (the index) is **never synced** — it's regenerated on each machine from the files that are actually present, so local memory titles never leak across machines.
- Toggle anytime: `memnir share <id>` / `memnir local <id>`.

Sync is filtered with `rsync --files-from=<list of scope:shared files>` — local files are never transmitted.

## Sync design

- `rsync -auz`: `-u` skips files newer on the receiver (**newest-wins**); **no `--delete`**, so files are never removed across machines (safe, but deletions must be done on both sides).
- Whatever was in a project's `memory/` before linking is backed up as `memory.bak.<ts>`.
- Logs at `~/.claude/memnir.log`.

## Requirements (macOS)

> ⚠️ Tested and supported on **macOS only** — relies on macOS/BSD behavior (`hostname -s`, BSD `sed -i ''`, `~/.zshrc`, the Tailscale.app path, Remote Login). Linux/Windows not yet supported.

| need | notes |
|---|---|
| **macOS** | Intel or Apple Silicon |
| **Rust / cargo** | to build (`rustup` or `brew install rust`) — or copy a prebuilt binary between same-arch machines |
| **zsh** | default macOS shell; the `memnir` alias goes in `~/.zshrc` |
| **rsync + ssh** | ship with macOS |
| **python3** | used by `install.sh` to merge hooks into `settings.json` (ships with Command Line Tools — `xcode-select --install`) |
| **[Tailscale](https://tailscale.com)** | the **Mac app** on both machines, on the same tailnet |
| **Remote Login** | enable on the machine you sync *to*: System Settings → General → Sharing → Remote Login (enable on both for two-way) |
| **SSH key auth** | passwordless between machines (`ssh-keygen` + the public key in the other machine's `~/.ssh/authorized_keys`) |
| **peer** | `~/.claude/memnir.conf` or env `MEMNIR_PEER` = `user@tailscale-host` of the other machine |

**macOS notes:**
- `systemsetup -setremotelogin on` needs the terminal to have Full Disk Access — the GUI toggle (Sharing) is easier and needs no FDA.
- Tailscale **SSH server (`tailscale up --ssh`) is Linux-only** — on macOS use regular Remote Login (OpenSSH `sshd`).

## Project layout

- [`src/main.rs`](src/main.rs) — all of Memnir (pure std)
- [`Cargo.toml`](Cargo.toml)
- [`install.sh`](install.sh) — build + bootstrap a machine into the mesh
- `.gitignore` (ignores `/target`, `dashboard.html`, `*.log`)

## License

[MIT](LICENSE)
