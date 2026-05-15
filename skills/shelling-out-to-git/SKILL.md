---
name: shelling-out-to-git
description: Non-obvious considerations when writing logic that shells out to Git from an app/lib.
---

It's hard to shell out to Git securely, so it's important to think carefully whenever you do.

## Config

Git reads `.git/config` from the local directory, so you need to know that that file doesn't exist/isn't malicious.
(For example, `git commit` with CWD set to an arbitrary attacker-controlled directory is unsafe if `.git/config` might already exist.)

Set `GIT_CONFIG_NOSYSTEM=1`, `GIT_CONFIG_GLOBAL=/dev/null`, and `GIT_CONFIG_COUNT=0` and unset `GIT_CONFIG_PARAMETERS`, and set `HOME=/dev/null`, to prevent loading system/user config.

## Clone/fetch targets

You're probably taking a URL as input to decide what to clone.
As early as possible, strip credentials (such as a username) from it, and preferably use the type system to identify that creds have been stripped: it's otherwise very easy to accidentally allow creds to appear in logs.
Default to only accepting `http` and `https` schemes unless you have evidence that other schemes will be required; leave the door open for other schemes, but it's generally easier to validate security when HTTP(S) are the only schemes implemented.

## Environment

Always carefully control the environment of the Git process we're creating, and default to completely stripping it and also running from `/` to ensure we're not already in a Git repo when running.
Otherwise, e.g. if the user has set a custom Git config file path, important assumptions we make about Git's behaviour might break.

(Resolve `git` from `$PATH` *before* clearing the environment.)

## Credentials

Never embed credentials in a remote URL, because Git is liable to persist them to disk and to echo them in error messages.

For HTTPS, pass creds through the process env:

```bash
GIT_TOKEN=… \
GIT_TERMINAL_PROMPT=0 \
git \
  -c credential.helper= \
  -c credential.helper='!f() { test "$1" = get && printf "username=foo\npassword=%s\n" "$GIT_TOKEN"; }; f' \
  clone https://github.com/foo/bar
```

* It's important to clear `credential.helper` first, because otherwise they stack.
* Adjust to pass in the correct username as appropriate (a GitHub PAT will work in both fields, if the remote is GitHub).
* You should make sure `GIT_TOKEN` doesn't contain `printf` metacharacters like newlines.

## `git` binary lifecycle

Make sure `git` has quit within some bounded scope.
(For example, if in Rust, `tokio::process::Command` needs `kill_on_drop(true)`.)

## Repo identification

Git permits `-C <path>`, but this will locate a `.git` directory that's been planted in `<path>` even if you intended `<path>` to be bare.
Use `--git-dir <path>` instead, to suppress discovery (remember you need `--work-tree` in that case too, for non-bare repos).

## Bare repo identification

*All* of the following need to be true for the repo to be definitely a bare repo confined to that directory:

* `HEAD` exists
* `core.bare` is configured `true`
* there are no symlinks at or transitively below the `objects/` or `refs/` directories
* there's no `commondir` (linked worktree gitdir)

Remember also that Git config has last-wins semantics: a first-match parser is wrong.

## Alternates

It's so confusing to allow usage of the Git alternates mechanism.
Don't do it, and if you want to be sure a write has taken place, you need to check that `objects/info/alternates` doesn't exist.

## Git notes footguns

* The default `-F` runs `stripspace` and silently mutates the bytes written. You should instead use `--no-stripspace` (and consider `--allow-empty`) unless you have some reason to use the default.
* Git note creation is not atomic. Concurrent writers against the same note will silently lose writes, because registering a Git note is simply force-replacing a commit, and there's no `--force-with-lease`-equivalent-style flag to ensure you are replacing what you thought you were replacing. This is frankly incredible. You need to plan around this.
* Creating a Git note creates a commit, so it requires `GIT_AUTHOR_*` and `GIT_COMMITTER_*` to be set (or some other way of obtaining identity, like config).

## Object format

Git supports `extensions.objectformat` of `sha256`, which your application might not; consider whether you need to check the object format before consuming objects.
