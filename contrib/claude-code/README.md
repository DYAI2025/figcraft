# FigCraft with Claude Code (macOS)

Setup for using FigCraft as the default wireframe and design tool from Claude Code in every
project: the signed release installed once, both MCP backends registered at user scope, and a
launcher that starts the app with its control channel so an agent can edit the open file live.

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
