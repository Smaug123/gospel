---
name: shelling-out-to-git
description: Non-obvious considerations when writing logic that shells out to Git from an app/lib.
---

It's hard to shell out to Git securely, so it's important to think carefully whenever you do.

## Config

Git reads `.git/config` from the local directory, so you need to know that that file doesn't exist/isn't malicious.
(For example, `git clone` with CWD set to an arbitrary attacker-controlled directory is unsafe if `.git/config` might already exist.)

Set `GIT_CONFIG_NOSYSTEM=1`, and set `HOME=/dev/null`, to prevent loading system/user config.

## Clone/fetch targets

You're probably taking a URL as input to decide what to clone.
As early as possible, strip credentials (such as a username) from it, and preferably use the type system to identify that creds have been stripped: it's otherwise very easy to accidentally allow creds to appear in logs.
Default to only accepting `http` and `https` schemes unless you have evidence that other schemes will be required; leave the door open for other schemes, but it's generally easier to validate security when HTTP(S) are the only schemes implemented.

## Environment

Always carefully control the environment of the Git process we're creating, and default to completely stripping it.
Otherwise, e.g. if the user has set a custom Git config file path, important assumptions we make about Git's behaviour might break.

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
