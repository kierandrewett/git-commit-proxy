# Git Commit Proxy

Git Commit Proxy checks the identity Git is about to write or publish. It lets Git choose an identity from its normal settings, then permits the operation only when the exact **name and email pair** is on your allow-list.

This helps prevent tools or scripts from leaving unexpected identities such as `Codex <gnoblin@localhost>` in commits and patches.

## Requirements

- Git
- Python 3.10 or later

The proxy uses only Python's standard library.

## Install

Install the executable as `git` in `~/.local/bin`:

```sh
mkdir -p ~/.local/bin
install -m 755 git ~/.local/bin/git
```

Put `~/.local/bin` before the system Git directory in your `PATH`. For zsh, add this to `~/.zshrc`; for Bash, add it to `~/.bashrc`:

```sh
export PATH="$HOME/.local/bin:$PATH"
```

Open a new shell, then confirm that `command -v git` prints `~/.local/bin/git` (or its full path). The proxy finds the real Git executable later in `PATH`.

If the real Git executable cannot be found automatically, set its path explicitly:

```sh
export GIT_COMMIT_PROXY_REAL_GIT=/usr/bin/git
```

## Set up the allow-list

Create the configuration file:

```sh
git proxy init
```

Edit `~/.config/git-commit-proxy/config.json` and add the exact name and email pairs you want Git to use:

```json
{
  "version": 1,
  "identities": [
    { "name": "Alex Example", "email": "alex@example.com" },
    { "name": "Alex Example (Work)", "email": "alex@work.example" }
  ]
}
```

Each entry is one permitted pair. A permitted name combined with a different permitted email is **not** permitted unless that pair has its own entry. Keep the config file private; it is created with your normal user permissions.

Git still chooses which identity to use. For example, set a default identity globally:

```sh
git config --global user.name "Alex Example"
git config --global user.email "alex@example.com"
```

You can also select another listed pair with repository config, `git -c user.name=...`, or Git's author and committer environment variables. The proxy checks the resulting identity when an operation would create or publish it; it does not bind identities to repository folders.

Check the current effective identities at any time:

```sh
git proxy status
```

It reports both the effective author and committer identities, and whether each is allowed.

## What is checked

The proxy checks identity-bearing operations, including commits, merge commits, annotated tags, stashes, notes, and relevant history rewrites. It checks source authors before cherry-pick, rebase, and `git am`; checks identities in `git fast-import`; and checks authors before `git format-patch` or `git send-email` publishes patches. It also checks `--author` values and prevents `git config` from setting `user.name` or `user.email` to values absent from the allow-list.

For example, with the sample allow-list above, this commit is allowed:

```sh
git -c user.name="Alex Example" -c user.email=alex@example.com commit -m "Update docs"
```

This one is blocked before Git runs the commit:

```sh
git -c user.name=Codex -c user.email=gnoblin@localhost commit -m "Update docs"
```

The same policy applies to commits started by tools that invoke `git` through your `PATH`.

## Configuration

The default file is `~/.config/git-commit-proxy/config.json`. To use another file, set `GIT_COMMIT_PROXY_CONFIG`:

```sh
export GIT_COMMIT_PROXY_CONFIG="$HOME/.config/my-git-identities.json"
```

The config must be JSON with `"version": 1` and an `"identities"` list.

## Troubleshooting

- **`git proxy init` says the configuration already exists:** edit the existing config; init does not overwrite it.
- **An expected identity is blocked:** compare the exact name and email shown by `git proxy status` with one complete pair in the JSON file. Names and emails are case-sensitive.
- **The proxy cannot find Git:** set `GIT_COMMIT_PROXY_REAL_GIT` to the real executable path.
- **A tool still writes an unexpected identity:** check that the tool uses `git` from `PATH`, and that `~/.local/bin` comes before the system Git directory. Calling the real Git binary directly bypasses the proxy.

## Limits

This is a PATH-based safeguard. A process with access to your account can bypass it by calling the real Git executable by absolute path, changing `PATH`, or writing Git objects with another program. Put the proxy first in `PATH` for shells and tools that run Git. It does not provide operating-system-level enforcement.
