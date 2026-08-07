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

### 2026-07-25 — Bandit levels 3-5

- **Level 3**: the password was in `inhere`, which looked empty under a plain `ls`. Linux hides any file starting with `.` (a dotfile) from the default listing — `ls -a` reveals them, including the two special entries every directory has: `.` (reference to the current directory) and `..` (reference to the parent directory, one level up — not a fixed "home", it's relative to wherever you are). Found `...Hiding-From-You`, read it the same way as level 1's leading-dash file: `cat ./...Hiding-From-You`.
- **Level 4**: 10 candidate files (`-file00` to `-file09`), only one human-readable. Opening each blindly risks dumping binary garbage to the terminal, so used `file ./*` to inspect content/type without opening anything. Ruled out `OpenPGP Secret Key` (a structured crypto format, not prose) and `Non-ISO extended-ASCII text, with NEL line terminators` (technically text, but a non-standard encoding not guaranteed to render cleanly) in favor of the one flagged as plain `ASCII text` — the simplest, most universal text encoding.
- **Level 5**: password buried somewhere under `inhere`'s subdirectories, defined only by properties (human-readable, exactly 1033 bytes, not executable) — manually walking the tree doesn't scale. `find` searches a whole directory tree by combinable criteria: `find ~/inhere -type f -size 1033c ! -executable`. Each flag is independent and swappable — e.g. dropping `-size 1033c` removes the size filter, flipping `! -executable` to `-executable` searches for executables instead.

**Takeaway:** all three levels are the same underlying lesson — Linux's default command output never shows everything that exists; it's incomplete by design, not by accident. `ls` hides dotfiles unless told `-a`, a plain look at a file says nothing about its real type until `file` inspects it, and manual browsing doesn't scale once criteria get specific enough to need `find`. The reflex going forward: when something "isn't there," ask what the default output is choosing not to show, not whether it exists.

Next: Bandit level 6+.

### 2026-08-06 — Bandit levels 6-7

- **Level 6**: password somewhere on the *entire* server, owned by user `bandit7`, group `bandit6`, exactly 33 bytes — `find` with no path defaults to searching from the current directory, so "somewhere on the server" means giving it the root explicitly: `find / -user bandit7 -group bandit6 -size 33c`. Same combinable-filter idea as level 5, extended with two new independent filters (`-user`, `-group`) — each one narrows the candidate list further; with only the size filter there could be several 33-byte files, adding owner and group is what narrows it to exactly one. Searching from `/` without permission to read every directory floods stderr with `Permission denied` — harmless noise, silenced with `2>/dev/null`, doesn't affect the real match on stdout.
- **Level 7**: password sits in `data.txt`, next to the word `millionth`. First instinct was `find data.txt -string millionth` — doesn't exist, because `find` only ever filters by filesystem metadata (name, size, owner, type), it never looks *inside* a file's content. The right tool for content is `grep <pattern> <file>`, which searches line by line for a match and prints it — functionally the command-line equivalent of Ctrl+F. `find` locates the right file; `grep` locates the right line inside it.

**Takeaway:** `find` and `grep` split the search space cleanly — one operates on filesystem metadata, the other on file contents — and neither substitutes for the other. When "where is this file" turns into "what's inside this file," that's the signal to switch tools, not to force more flags onto `find`.

Next: Bandit level 8+.

### 2026-08-07 — Bandit level 8

- **Level 8**: password is the *only* line in `data.txt` that occurs exactly once — every other line repeats. First attempt was `sort data.txt | uniq --count`, then scanning the output by eye for the line with `1` next to it — worked, but only because the list was small enough to eyeball; it doesn't scale to a file with thousands of lines. The actual lesson: `uniq` never compares the whole file, only each line against the one *immediately before it* — it has no memory further back. Two identical lines that aren't adjacent are invisible to it as duplicates. `sort` isn't cosmetic here — it's what physically moves every identical line next to its twin, which is the only way `uniq`'s adjacent-only comparison can catch them. `uniq -u` (`--unique`) then filters automatically for lines with zero repeats, no manual scanning needed: `sort data.txt | uniq -u`.

**Takeaway:** a recurring shape in Unix text tools — many process input one line at a time with no global memory (streaming, single-pass). Whenever the task needs to compare or group things that aren't already adjacent, `sort` is usually the step that makes them adjacent first. Same pattern shows up again with tools like `join`.

Next: Bandit level 9+.
