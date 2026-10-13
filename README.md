# Pinboard for herdr

A note for each [herdr](https://herdr.dev) workspace: one short line on the top border of every
pane in it, and the whole note in a drawer at the top of each tab, opened and closed with a key.

## Why

You step away from a workspace for a while. When you come back, you want the same set of context
waiting for you every time, from you or from the agent working there: what the blocker was, a
summary that always has the same layout, a to-do list. Whatever you want, really. The pinboard
is where that goes. Agents keep it current as they work (see [For agents](#for-agents)), and a
glance at any pane's border tells you what the workspace is for.

```
╭─ 📌 pinboard · Ctrl+B then m closes ─────────────────────────────────────────╮
│  BLOCKER: waiting for the staging database to come back                      │
│                                                                              │
│  Payments retry rework                                                       │
│  Done: retries with backoff. Next: the dead-letter queue, then load tests.   │
╰──────────────────────────────────────────────────────────────────────────────╯
╭─ 📌 BLOCKER: waiting for the staging database to come back · Payments retr… ─╮
│ $ claude                                                                     │
│                                                                              │
```

The top box is the drawer, at the top of the tab. Every other pane in the workspace carries the
one-line note on its border, with the blocker first while there is one.

## What it needs

- herdr 0.9.1 or later, on Linux
- bash, `jq`, Python 3.11 or later, `flock` (util-linux) and `less`

macOS isn't supported yet: the command uses Linux's `flock` and `stat`.

## Install

```sh
herdr plugin install drewdunne/herdr-pinboard
```

To type `pinboard` anywhere, link the command into a folder on your `PATH`:

```sh
mkdir -p ~/.local/bin
ln -s "$(herdr plugin list --json | jq -r '.result.plugins[] | select(.plugin_id == "drewdunne.pinboard") | .plugin_root')/bin/pinboard" ~/.local/bin/pinboard
```

Then set herdr up as below, and set a note in any workspace: `pinboard set "What this workspace is for"`.

## herdr settings it needs

In herdr's config (`~/.config/herdr/config.toml`):

```toml
[ui]
# Frame single panes too, so every pane has a top border to carry the note.
pane_borders = "always"

# Keys: prefix then m opens or closes the drawers; prefix then Shift+M shows the whole note.
[[keys.command]]
key = "prefix+m"
type = "plugin_action"
command = "drewdunne.pinboard.drawer"

[[keys.command]]
key = "prefix+shift+m"
type = "plugin_action"
command = "drewdunne.pinboard.whole"
```

Then run `herdr server reload-config`, or restart herdr. The drawer's border names the key you
bound ("Ctrl+B then m closes" with herdr's default prefix), read from this config.

**Attaching from another machine** (`herdr --remote`): keys bound to plugin actions only work
with `--remote-keybindings server`, which uses the server's keys. With herdr's default (the
attaching machine's keys) every key bound to a command is dropped.

```sh
herdr --remote my-server --remote-keybindings server
```

## Commands

| Command | What it does |
|---|---|
| `pinboard set "<line>"` | Set this workspace's note. Options: `--workspace ID` for another workspace, `--full-file FILE` for a longer note shown in the drawer, `--if-unset` to leave an existing note alone. |
| `pinboard show` | Print it (`--workspace ID` too). |
| `pinboard clear` | Remove it. |
| `pinboard block "<what's blocking you>"` | Put `BLOCKER: …` at the top of the border and the drawer. `pinboard unblock` takes it off. |
| `pinboard drawer [open\|close\|toggle]` | The drawers. `toggle`, the key's action, closes them all if this tab has one, and otherwise opens this tab's, and then each other tab's as you visit it. A drawer you close by hand (`q`) stays closed until the next toggle. |
| `pinboard whole` | The whole note in a popup over this pane. It scrolls; `q` closes it. |
| `pinboard check [--hint TEXT]` | For an agent's prompt hook: a reminder to set a note when the workspace has none, and nothing otherwise. `--hint` adds your own conventions. |
| `pinboard pause`, `pinboard resume` | Around an update; see [Updating](#updating). |
| `pinboard refresh` | Relabel every pane from the notes, and close drawers that are no longer working. |
| `pinboard restore` | Put every note back after a restart. herdr runs it when it starts. |

Inside a herdr pane the command works on that pane's workspace and session. From outside herdr,
put `--session NAME` first (`pinboard --session work set --workspace w2 "…"`).

Notes are kept in files, in herdr's state folder for the plugin, so they survive restarts.

## For agents

Ask your agents to keep the pinboard current: set it when they start, update it when something
changes, and block or unblock it when you're waiting on something. A prompt hook reminds them when
a workspace has none. For Claude Code, in `~/.claude/settings.json`:

```json
{
  "hooks": {
    "UserPromptSubmit": [
      { "hooks": [{ "type": "command", "command": "pinboard check --hint 'Start the line with the ticket number.'", "timeout": 5 }] }
    ]
  }
}
```

## Updating

Installing again replaces herdr's copy and registers the plugin again, which would orphan open
drawers and switch the plugin on if you'd switched it off. So pause around it:

```sh
pinboard pause
herdr plugin install drewdunne/herdr-pinboard     # or --ref <tag or commit> for a set version
pinboard resume
```

`pause` closes the drawers (they stay switched on) and stops the plugin touching any pane, in
every session, until `resume`. Notes you set meanwhile are kept, and show on `resume`. `resume`
puts the plugin back on or off as it was, runs the start-up step for any session that started
during the pause, relabels every pane and reopens this tab's drawer.

## Settings

Optional, in `pinboard.env` in the plugin's config folder (`herdr plugin config-dir
drewdunne.pinboard` prints where that is). Plain `KEY=value` lines; a value with spaces goes in
quotes, and `~/` means your home folder.

| Setting | Default | What it's for |
|---|---|---|
| `PINBOARD_STATE_DIR` | herdr's state folder for the plugin | Where notes are kept: a folder per herdr session inside it. |
| `PINBOARD_TITLE_SOURCE` | `pinboard` | The name pane titles are reported to herdr under. |
| `PINBOARD_KEY_ACTIONS` | empty | Other action names whose keys open the drawer or the whole note, each ending in `.drawer` or `.whole`, so the hints find them. |

These are for moving over from another tool without copying anything: point the pinboard at
that tool's notes folder (same layout: `<session>/<workspace>.line`, `.full`, `.blocker`, and a
`lock` file both take), report titles under its name so no old title is left behind, and keep
the keys you already have. While another tool still has the folder, an installer can write
`paused` in it (holding `enabled=true` or `enabled=false`) before the first install, so the
plugin starts paused, and run `pinboard resume` after.

## Limits

- A border shows 80 characters, so a longer line is cut at a word.
- A drawer is at least 10% of the tab (herdr's smallest split) and sits above the tab's first pane.
- Another plugin that writes pane titles covers the line while its title is newer.

## Tests

```sh
tests/run
```

The tests run a throwaway herdr in a scratch folder, with every `HERDR_` variable unset and their
own home, config, state and plugin list. They check that no running session is visible before
registering anything, so they never touch the herdr you're using.

## License

MIT
