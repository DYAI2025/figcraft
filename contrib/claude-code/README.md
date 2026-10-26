# FigCraft with Claude Code and other agents

Setup for using FigCraft as the default wireframe and design tool from Claude Code in every
project: the signed release installed once, both MCP backends registered at user scope, and a
launcher that starts the app with its control channel so an agent can edit the open file live.
Sections 5 and 6 add a Linux host that other agents (Codex, GPT-based agents) drive over SSH, and a
collaboration server for one shared live file.

Verified on macOS (Apple Silicon) with FigCraft v0.11.2 and Claude Code, 2026-10-25.

## 1. Install the release

Use the signed and notarized universal builds from the
[GitHub release](https://github.com/storytold/figcraft/releases) and check them against
`SHA256SUMS.txt`:

```sh
V=0.11.2
D=$(mktemp -d)
gh release download "v$V" -R storytold/figcraft -D "$D" \
  -p "figcraft-$V-macos-universal.dmg" -p "figcraft-cli-$V-macos-universal.zip" -p SHA256SUMS.txt
(cd "$D" && grep macos SHA256SUMS.txt | shasum -a 256 -c -)

# App
hdiutil attach -nobrowse -readonly -mountpoint "$D/mnt" "$D/figcraft-$V-macos-universal.dmg"
ditto "$D/mnt/FigCraft.app" /Applications/FigCraft.app
hdiutil detach "$D/mnt"

# CLI at a versionless path, so the MCP config never changes on updates
unzip -q "$D/figcraft-cli-$V-macos-universal.zip" -d "$D/cli"
mkdir -p ~/.local/opt/figcraft ~/.local/bin
ditto "$D/cli/figcraft-cli-$V-macos-universal" ~/.local/opt/figcraft
echo "$V" > ~/.local/opt/figcraft/VERSION
ln -sf ~/.local/opt/figcraft/figcraft-cli ~/.local/bin/figcraft-cli

# Launcher (this folder)
install -m 755 contrib/claude-code/figcraft-open ~/.local/bin/figcraft-open
```

Optional checks: `spctl -a -t exec -vv /Applications/FigCraft.app` reports
`source=Notarized Developer ID`, and `figcraft-cli --version` prints the version.

To update, repeat the steps with the new version: files are replaced in place, the MCP
registration stays as it is.

## 2. Register the MCP servers (all projects)

```sh
claude mcp add -s user figcraft -- ~/.local/opt/figcraft/figcraft-cli mcp
claude mcp add -s user figcraft-app -- ~/.local/opt/figcraft/figcraft-cli mcp --attach 7979
```

| Server | Backend | Use it for |
|---|---|---|
| `figcraft` | headless, in-process document, no window | quick drafts, exports, CI-style renders |
| `figcraft-app` | the running app over its loopback control channel | working on a file live, in front of the user |

`figcraft-app` only connects while the app runs with `--control 7979`. Start it with the launcher:

```sh
figcraft-open                    # empty session
figcraft-open design/login.figcraft
```

The launcher exits early when the control channel is already listening, and refuses (exit 1) when
FigCraft runs without it: quit the app first, then start it again through the launcher. The port
comes from `FIGCRAFT_CONTROL_PORT` (default 7979; the MCP registration must use the same port).

Both servers are stdio processes. The headless one is unconfined by default; to keep an agent to
one folder, add `--automation-read-root DIR --automation-write-root DIR` (see
[`docs/mcp.md`](../../docs/mcp.md#confining-file-access)).

## 3. Things agents get wrong

Measured while setting this up:

- `create_node`'s `parent` takes the numeric node **id** returned by the call that created the
  parent. A layer name is rejected: `` `parent` must be a node id (got "Login wireframe") ``.
- `export` and most commands act on the **selection** when `ids` is left out, and every new node
  becomes the selection. Exporting right after building a frame without `ids` exports the last
  child, not the frame: pass `ids` (or `figcraft-cli export FILE -o out.png --node ID`).
- Check the result before calling a design done: `screenshot` renders the page or given layers;
  attached, `screenshot {"window": true}` captures the real app window.

Templates are a good starting point: `run_command template.list`, then
`run_command file.newFromTemplate {"template": "<id>"}` (one of them is a wireframe kit).

## 4. Global instruction snippet

Add something like this to `~/.claude/CLAUDE.md` so every session reaches for FigCraft first:

```markdown
## Wireframes & UI Design — FigCraft (preferred tool)

FigCraft is the preferred tool for wireframes and design work in every project. Use the
`figcraft-app` MCP (start the app with `figcraft-open [file]`) to work live, or `figcraft`
(headless) for drafts and exports. Keep `.figcraft` files in the project's `design/` folder.
`create_node` `parent` is a node id; pass `ids` to `export`; verify with `screenshot`.
```

Computer use also works on the app (native window) for gestures the commands don't cover, but the
MCP is faster and exact: every menu item and tool is one of 400+ commands (`list_commands`).

## 5. Other agents: headless MCP on a Linux host over SSH

FigCraft's MCP speaks stdio only. Any MCP client that can start a command (Claude Code, Codex,
other GPT-based agents) reaches a Linux host by starting `figcraft-cli mcp` there through `ssh`:
the SSH session is the transport, nothing listens on the network, and SSH keys are the
authentication.

Verified on Arch Linux x86_64 with v0.11.2, 2026-10-26: `initialize` (server `figcraft`
0.11.2, protocol `2025-06-18`), `tools/list` (18 tools), a frame with text and a rectangle built,
exported as a 720×400 PNG and saved as `.figcraft` inside the design folder, and an export to a path
outside it refused (`automation path rejected: … is outside the folders that may be written`).

### Install on the host

Use the tarball (no package manager footprint), check it against `SHA256SUMS.txt`, and keep a
versionless path for the MCP configs:

```sh
V=0.11.2
D=~/.cache/figcraft-dl/$V; mkdir -p "$D" && cd "$D"
B=https://github.com/storytold/figcraft/releases/download/v$V
for f in SHA256SUMS.txt figcraft-$V-linux-x86_64.tar.gz; do curl -fsSLO "$B/$f"; done
grep "  figcraft-$V-linux-x86_64.tar.gz$" SHA256SUMS.txt | sha256sum -c -

O=~/.local/opt/figcraft
mkdir -p "$O/releases/$V" ~/.local/bin ~/design/figcraft
tar xzf figcraft-$V-linux-x86_64.tar.gz -C "$O/releases/$V" --strip-components=1
ln -sfn "releases/$V" "$O/current"
ln -sfn ../opt/figcraft/current/bin/figcraft-cli ~/.local/bin/figcraft-cli
figcraft-cli --version
```

Updating is the same with the new version; `current` moves, the MCP configs stay. Use
`linux-aarch64` on ARM hosts (`uname -m`).

### Register it in a client

Always pass the automation roots: the headless server otherwise reads and writes anything the SSH
user can. Use absolute paths (a non-interactive SSH command gets no login `PATH`), and a host alias
that logs in without prompts (`BatchMode`; key without passphrase or loaded in an agent).

```sh
# Claude Code (any machine with the alias `arch` in ~/.ssh/config)
claude mcp add -s user figcraft-arch -- ssh -T -o BatchMode=yes arch \
  /home/ben/.local/bin/figcraft-cli mcp \
  --automation-read-root /home/ben/design/figcraft --automation-write-root /home/ben/design/figcraft
```

```toml
# Codex (~/.codex/config.toml)
[mcp_servers.figcraft-arch]
command = "ssh"
args = ["-T", "-o", "BatchMode=yes", "arch", "/home/ben/.local/bin/figcraft-cli", "mcp",
        "--automation-read-root", "/home/ben/design/figcraft",
        "--automation-write-root", "/home/ben/design/figcraft"]
```

```json
{ "mcpServers": { "figcraft-arch": { "command": "ssh", "args": ["-T", "-o", "BatchMode=yes", "arch",
  "/home/ben/.local/bin/figcraft-cli", "mcp",
  "--automation-read-root", "/home/ben/design/figcraft",
  "--automation-write-root", "/home/ben/design/figcraft"] } } }
```

An agent running on the host itself drops the `ssh … arch` part. The server's stderr names the
roots it enforces (`file access confined to {"read":[…],"write":[…]}`); check it once.

Each SSH session is its own headless document: work that should outlive the session goes into a
`.figcraft` file with `save_file` (inside the write root), or into a shared file on a collaboration
server (next section). When SSH runs through a `cloudflared access ssh` ProxyCommand, the Access
token expires and the next start needs a browser login; agents without a browser then fail to
connect, so prefer a LAN or tailnet alias for unattended agents.

## 6. One live file for the app, the browser build and agents

`figcraft-server` (see [`docs/server.md`](../../docs/server.md)) keeps a file on a server that
the desktop app, the browser build and headless agents edit together. For a personal server there is
no need to expose it: run it on the host's loopback (`127.0.0.1:47900`, the default, with
`open_publish` off so only the server key publishes) and forward the port wherever a client runs:

```sh
ssh -N -L 47900:127.0.0.1:47900 arch      # keep running while you work
curl -fs http://127.0.0.1:47900/healthz   # 200 ok through the tunnel
```

Links the server prints name `ws://127.0.0.1:47900/…`, which is correct on every client that
forwards the same local port. On the host, the server key comes from
`figcraft-server --data <data dir> key` (run as the server's user); give it only to whoever may
publish. Then:

- **Desktop app:** File ▸ Publish to server…, address `ws://127.0.0.1:47900/?key=<key>`; share the
  edit link or a join code.
- **Agents (MCP):** `run_command collab.join {"link": "<edit link>"}` (or `{"code": "…"}`) joins the
  same file; every change shows up live in the app.

The upstream `figcraft-server.service` in the release archive is a hardened systemd unit for this;
keep its `IPAddressAllow=localhost` / `IPAddressDeny=any` lines when the server stays on loopback.
