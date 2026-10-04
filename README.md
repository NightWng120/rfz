# rfz

Fuzzy-find files on a remote machine over SSH, pick what you want, and transfer
just those with rsync. One bash script, no daemon, no config. This was created by Claude AI

```
rfz myserver:/srv/media/ ~/Downloads
```

## Features

- **Three ways to pick files**: fuzzy search over a full listing, live
  server-side search for huge trees, or a directory browser.
- **Multi-select** files and directories (selecting a directory transfers all of it).
- **One SSH connection** is opened and reused for listing, previews and the
  transfer (`ControlMaster`/`ControlPersist`), so there is one handshake.
- **Fast listing**: runs `fd` on the server when available, falls back to POSIX `find`.
- **Remote preview** pane (directory listings, first 4 KB of text files).
- Works with **`~/.ssh/config` aliases**, so `myserver:` replaces `user@ip`.
- Dry run, excludes, and passthrough of any extra rsync flags.
- Works in **Termux** on Android.

## Requirements

| Where  | Needs |
| ------ | ----- |
| Client | `bash`, `ssh`, `rsync` (3.1+ for progress output), `fzf` |
| Server | `sh`, `find`, and an SSH login. `fd` or `fdfind` is optional but faster |

fzf versions: default mode works with any recent fzf, live mode (`-l`) needs
0.38+, and browse mode (`-b`) needs 0.45+.

## Install

```bash
git clone <your-repo-url> && cd <repo>
chmod +x rfz
cp rfz ~/.local/bin/        # or /usr/local/bin, or $PREFIX/bin on Termux
```

Termux:

```bash
pkg install openssh rsync fzf fd
```

## Usage

```
rfz [options] SOURCE DEST [-- extra rsync args]
```

`SOURCE` is a local directory or `[user@]host:path`. `DEST` can be local or
remote. Paths starting with `~` or relative paths resolve against the remote
home directory, and `host:` alone means the remote home.

```bash
rfz myserver:/srv/media/ ~/dl                 # full listing, fuzzy filter
rfz -l myserver:/srv/media/ ~/dl              # live server-side search
rfz -b myserver:~ ./home-copy                 # browse directories
rfz -n -x .git -x node_modules myserver:repo/ ./repo   # dry run with excludes
rfz myserver:/data/ /mnt/backup -- --bwlimit=2000      # extra rsync flags
```

### Options

| Option | Description |
| ------ | ----------- |
| `-l` | Live mode: search on the server per keystroke (best for huge trees) |
| `-b` | Browse mode: navigate directories one level at a time |
| `-s` | Shallow: list only the top level of SOURCE (default mode) |
| `-n` | Dry run (`rsync --dry-run --itemize-changes`) |
| `-y` | Skip the confirmation prompt |
| `-I` | Respect `.gitignore`/`.ignore` files when listing (needs `fd`) |
| `-x PATTERN` | Exclude glob, repeatable (applies to listing and transfer) |
| `-q QUERY` | Initial fzf query |
| `-e CMD` | SSH command, e.g. `-e "ssh -p 2222 -i ~/.ssh/key"` |
| `-P` | Disable the preview pane |
| `-M` | Disable SSH connection multiplexing |
| `-h` | Help |

## Keys

**Default mode**

| Key | Action |
| --- | ------ |
| Tab / Shift-Tab | Select / deselect and move |
| Ctrl-A / Ctrl-D | Select all matches / clear selection |
| Enter | Confirm |

**Live mode (`-l`)**

| Key | Action |
| --- | ------ |
| Tab | Add to basket (survives new searches) |
| Ctrl-U | Clear basket |
| Enter | Finish: uses the basket, or the highlighted item if the basket is empty |

**Browse mode (`-b`)**

| Key | Action |
| --- | ------ |
| Tab | Mark / unmark the highlighted item (shown with `+`) |
| Enter | Transfer marked items, or the highlighted one if none are marked. On `../`, goes up |
| Right | Open directory |
| Left / Ctrl-B | Go up |
| Ctrl-U | Clear all marks |
| Esc | Quit without transferring |

Marks persist as you move between directories, so one transfer can combine
files from several folders.

## SSH setup

Key auth and a config alias make this painless:

```bash
ssh-keygen -t ed25519
ssh-copy-id user@host
```

`~/.ssh/config`:

```
Host myserver
  HostName 203.0.113.10
  User alice
  IdentityFile ~/.ssh/id_ed25519
  IdentitiesOnly yes
```

Then `rfz myserver:/path dest` just works. Put anything `-e` cannot express
(like `ProxyJump`) in the config file.

## How it works

1. Builds an `ssh` command with multiplexing and keepalive options.
2. Pipes a small listing script to `sh -s` on the source (over SSH for remote
   sources). It runs `fd`, or `find` if `fd` is missing.
3. Feeds the paths to `fzf`.
4. Writes your selection to a file and runs
   `rsync -ahr --files-from=FILE SOURCE/ DEST`, so selected paths keep their
   relative structure under `DEST`.

The multiplexed connection stays alive for 10 minutes after use. Close it
sooner with `ssh -O exit host`.

## Limitations

- Filenames containing newlines are not supported.
- `rsync://` daemon sources are not supported; use SSH.
- `[user@]host:path` only; bracketed IPv6 hosts are not handled.
- `rsync -e` splits on spaces, so `-e` values cannot contain quoted arguments.
  Use `~/.ssh/config` for complex options.
- The default full listing downloads every path before fzf opens. On very large
  trees use `-s`, `-l` or `-b`.
- Multiplexing relies on Unix sockets and OpenSSH, so it will not work on
  filesystems without socket support (e.g. shared storage on Android).

## Troubleshooting

- **Prompted for a password**: check key permissions (`~/.ssh` 700,
  `authorized_keys` 600) and run `ssh -v host`.
- **`cannot cd to ...`**: the path does not exist or is not readable by your SSH user.
- **Browse or live mode errors about unknown actions**: your fzf is too old; run
  `fzf --version`.
- **Ctrl-B does nothing in tmux**: it is the tmux prefix. Use the Left arrow or
  the `../` entry instead.
- **Stale connection after switching networks**: run `ssh -O exit host`.
- **Android kills the background SSH master**: run `termux-wake-lock` and
  disable battery optimization for Termux. A new master starts automatically.
