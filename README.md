# Learning Log

Documenting my path from Computer Engineering student to Platform Engineer / SRE — infrastructure, cloud, Linux, and everything in between. Reasoning over results: each entry captures what I learned and how I got there, not just the final answer.

## Entries

### 2026-07-15 — Environment setup

Set up the local development environment for the journey ahead:

- macOS terminal: Homebrew, iTerm2, Oh My Zsh, VS Code
- Docker Desktop, verified with `docker run hello-world`
- UTM + Ubuntu Server 26.04 (ARM64) VM (`platform-lab`), SSH access configured from host to guest
- GitHub: SSH authentication, this repository

Next: OverTheWire Bandit (Linux fundamentals).

### 2026-07-22 — Bandit levels 0-2

First real session on [OverTheWire Bandit](https://overthewire.org/wargames/bandit/) — connected over SSH and worked through the first three levels.

- **Level 0**: SSH connection basics. The first-ever connection to a new host asks you to accept its host key fingerprint — this is SSH's defense against man-in-the-middle attacks (trust-on-first-use); once accepted, the key is cached in `~/.ssh/known_hosts` and any future mismatch triggers a loud warning instead of silently connecting. Found the next password with the basic `ls` → `cat` pair.
- **Level 1**: the password file was named literally `-` (a single dash). `cat -` doesn't open a file named `-` — by long-standing Unix convention, many CLI tools read a bare `-` argument as "read from stdin" instead, which is why the command just hung waiting for keyboard input. Fix: `cat ./-` — prefixing with `./` turns it into a path, so it no longer matches the special bare-dash convention. The same problem (and the same fix) applies to any filename starting with `-`, since command-line tools generally parse leading-dash arguments as options/flags before treating them as filenames.
- **Level 2**: password file named `--spaces in this filename--` — stacked both problems from level 1 (leading dashes) and a new one (literal spaces in the name). First attempt (`cat ./--spaces in this filename--`) fixed the dash problem but not the spaces — the shell split it into 4 separate arguments and `cat` tried to open each one. Second attempt (`cat --spaces\ in\ this\ filename--`) fixed the spaces via backslash-escaping but not the leading dashes — the single resulting argument still started with `--`, so it got parsed as an attempted (invalid) long option. Final fix combined both: `cat ./--spaces\ in\ this\ filename--`.

**Takeaway:** shell argument-splitting (spaces) and a command's own flag-vs-filename parsing (leading `-`/`--`) are two independent layers a filename can trip — separately or at the same time. The fixes don't conflict; they compose.

Next: Bandit level 3+.
