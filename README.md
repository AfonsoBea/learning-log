# Learning Log

A record of my path from Computer Engineering student (FEUP, Porto) towards Platform Engineering and SRE, covering Linux, infrastructure and cloud.

Each entry documents the reasoning behind a solution, including the attempts that failed, rather than only the final answer.

## Progress

| Track | Status |
|---|---|
| Environment setup | Complete |
| [OverTheWire Bandit](https://overthewire.org/wargames/bandit/) | Levels 0–11 complete |

## Entries

| Date | Entry | Topics |
|---|---|---|
| 2026-07-15 | [Environment setup](#2026-07-15--environment-setup) | Homebrew, Docker, UTM, SSH |
| 2026-07-22 | [Bandit levels 0–2](#2026-07-22--bandit-levels-02) | SSH host keys, argument parsing |
| 2026-07-25 | [Bandit levels 3–5](#2026-07-25--bandit-levels-35) | Hidden files, `file`, `find` |
| 2026-08-06 | [Bandit levels 6–7](#2026-08-06--bandit-levels-67) | `find` from root, stderr redirection, `grep` |
| 2026-08-07 | [Bandit level 8](#2026-08-07--bandit-level-8) | `sort`, `uniq` |
| 2026-09-18 | [Bandit level 9](#2026-09-18--bandit-level-9) | Binary files, `strings`, pipes |
| 2026-09-18 | [Bandit level 10](#2026-09-18--bandit-level-10) | Base64, encoding vs encryption |
| 2026-09-18 | [Bandit level 11](#2026-09-18--bandit-level-11) | ROT13, `tr`, positional mapping |

---

## 2026-07-15 — Environment setup

Set up the local development environment:

- macOS terminal: Homebrew, iTerm2, Oh My Zsh, VS Code
- Docker Desktop, verified with `docker run hello-world`
- UTM with an Ubuntu Server 26.04 (ARM64) virtual machine (`platform-lab`), with SSH access from host to guest
- GitHub with SSH authentication, and this repository

**Next:** OverTheWire Bandit (Linux fundamentals).

---

## 2026-07-22 — Bandit levels 0–2

First session on [OverTheWire Bandit](https://overthewire.org/wargames/bandit/), connecting over SSH.

### Level 0: SSH basics

On the first connection to a new host, SSH asks you to accept its host key fingerprint. This is its protection against man-in-the-middle attacks (trust on first use). Once accepted, the key is stored in `~/.ssh/known_hosts`, and any later mismatch produces a warning instead of connecting silently. The password itself was found with `ls` and `cat`.

### Level 1: a file named `-`

`cat -` hung waiting for input. By Unix convention, many tools read a bare `-` as standard input rather than as a filename. Prefixing it with `./` turns it into a path:

```bash
cat ./-
```

The same applies to any filename starting with `-`, since tools parse leading dashes as options before treating an argument as a file.

### Level 2: `--spaces in this filename--`

This combined the leading-dash problem from level 1 with literal spaces in the name.

| Attempt | Command | Result |
|---|---|---|
| 1 | `cat ./--spaces in this filename--` | Fixed the dashes, but the shell split the name into four arguments |
| 2 | `cat --spaces\ in\ this\ filename--` | Fixed the spaces, but the argument was parsed as an invalid option |
| 3 | `cat ./--spaces\ in\ this\ filename--` | Worked |

**Takeaway:** shell word-splitting and a program's own option parsing are two separate layers. A filename can break either one, or both, and the fixes can be combined.

**Next:** Bandit level 3.

---

## 2026-07-25 — Bandit levels 3–5

### Level 3: hidden files

`inhere` looked empty with `ls`, because files starting with `.` are hidden from the default listing. `ls -a` shows them, along with the two entries every directory contains: `.` (the current directory) and `..` (its parent). The file was `...Hiding-From-You`:

```bash
cat ./...Hiding-From-You
```

### Level 4: the only human-readable file

There were ten candidate files and only one was human-readable. Rather than opening each one and risking binary output in the terminal, I used `file` to check their types:

```bash
file ./*
```

I ruled out `OpenPGP Secret Key` and `Non-ISO extended-ASCII text, with NEL line terminators`, and chose the one reported as plain `ASCII text`.

### Level 5: searching by properties

The password was somewhere under `inhere`, described only by its properties: human-readable, 1033 bytes, not executable. `find` can combine these as independent filters:

```bash
find ~/inhere -type f -size 1033c ! -executable
```

**Takeaway:** default command output is deliberately incomplete. `ls` hides dotfiles, a filename says nothing about the file's real type, and browsing by hand stops working once the criteria become specific. When something seems to be missing, the first question should be what the default view leaves out.

**Next:** Bandit level 6.

---

## 2026-08-06 — Bandit levels 6–7

### Level 6: searching the whole server

The password was somewhere on the server, owned by user `bandit7` and group `bandit6`, and exactly 33 bytes. Without a path, `find` searches from the current directory, so the search has to start at `/`. Size alone could match several files; adding owner and group narrows it to one.

Searching from `/` also produces many `Permission denied` errors. These go to stderr and can be discarded without affecting the results on stdout:

```bash
find / -user bandit7 -group bandit6 -size 33c 2>/dev/null
```

### Level 7: searching inside a file

The password was in `data.txt`, next to the word `millionth`. My first attempt, `find data.txt -string millionth`, does not exist: `find` filters on metadata (name, size, owner, type) and never reads file contents. `grep` searches contents line by line:

```bash
grep 'millionth' data.txt
```

**Takeaway:** `find` answers where a file is; `grep` answers what is inside it. When the question changes, the tool should change too.

**Next:** Bandit level 8.

---

## 2026-08-07 — Bandit level 8

The password was the only line in `data.txt` that appears exactly once.

My first attempt was `sort data.txt | uniq --count`, then looking for the line with a count of 1. It worked, but only because the output was short enough to read by eye.

The important detail is that `uniq` only compares each line with the one directly before it. Duplicates that are not adjacent go undetected. `sort` is therefore a precondition, not a cosmetic step: it places identical lines next to each other so that `uniq -u` can keep only the lines that never repeat.

```bash
sort data.txt | uniq -u
```

**Takeaway:** many Unix text tools process input in a single pass, one line at a time. When a task depends on comparing lines that are not adjacent, sorting first is usually what makes it possible.

**Next:** Bandit level 9.

---

## 2026-09-18 — Bandit level 9

The password was in `data.txt`, in one of the few human-readable strings, preceded by several `=` characters.

### First attempt

```bash
grep "=" data.txt
# grep: data.txt: binary file matches
```

`file data.txt` reports only `data`, meaning the file is binary. Every file is made of bytes; in a text file each byte corresponds to a printable character, while here most do not. When `grep` encounters a null byte it treats the file as binary and does not print the matching lines.

### Forcing text mode

After some research I added `-a`, which makes `grep` treat the file as text:

```bash
grep -a "=" data.txt
```

This found the password, but the output was mostly unreadable data that I had to scan manually, the same limitation as the first attempt in level 8.

### Extracting the readable parts first

`strings` outputs only sequences of four or more printable characters and discards everything else:

```bash
strings data.txt | grep "="
```

Here `grep` no longer reads `data.txt` directly. It reads the output of `strings`, which is plain text, so `-a` is no longer needed.

### Refining the pattern

This still returned 13 lines, of which only 4 were relevant. In random binary data a single `=` appears fairly often by chance, but several in a row almost never do. Matching two or more removed the noise:

```bash
strings data.txt | grep "=="
```

**Takeaway:** as in level 8, the solution is a pipeline in which each tool does one job: first prepare the data (`sort`, `strings`), then filter it (`uniq -u`, `grep`).

**Next:** Bandit level 10.

---

## 2026-09-18 — Bandit level 10

The password was in `data.txt`, which contains Base64-encoded data.

### What Base64 is

Base64 represents any sequence of bytes using only 64 characters: `A–Z`, `a–z`, `0–9`, `+` and `/`. It works in blocks of 3 bytes, each written as 4 characters, so encoded data is roughly a third larger than the original. The `=` at the end is padding, used when the input length is not a multiple of 3.

Its purpose is compatibility. Protocols such as email (SMTP), URLs, JSON and HTTP headers were designed to carry text, and raw bytes can be altered or misinterpreted in transit. Encoding them as Base64 lets binary data pass through these channels unchanged.

It provides no confidentiality. There is no key, and anyone can decode it with a single command. This matters in practice: Kubernetes Secrets, for example, store values in Base64 by default, which on its own does not make them secure.

### Solving the level

`file data.txt` reported `ASCII text`, so there was nothing to extract with `strings`. My first working solution was to copy the encoded string by hand into `echo`:

```bash
echo <encoded string> | base64 --decode
```

My attempt to avoid the copy, `echo data.txt | base64 --decode`, failed with `invalid input`: `echo` prints its argument literally, so `base64` received the text `data.txt` rather than the file's contents. Reading the file instead solved it:

```bash
cat data.txt | base64 --decode
```

`base64` also accepts a filename directly, which removes the need for `cat`:

```bash
base64 -d data.txt
```

A detail I had wrong in my notes: `echo -n` suppresses the trailing newline rather than adding one. `echo "Hello" | base64` encodes the newline as well and produces `SGVsbG8K`, while `echo -n "Hello" | base64` produces `SGVsbG8=`. The string in this level ended in `Cg==`, which is an encoded newline.

**Takeaway:** encoding and encryption solve different problems. Encoding makes data safe to transport; encryption makes it unreadable without a key. It also paid off to check the file type first: `file` showed plain text, so the extra filtering step from level 9 was unnecessary.

**Next:** Bandit level 11.

---

## 2026-09-18 — Bandit level 11

The password was in `data.txt`, with every letter rotated by 13 positions (ROT13).

### ROT13

ROT13 is a Caesar cipher with a fixed shift of 13: each letter is replaced by the one 13 places later in the alphabet, so `MARIA` becomes `ZNEVN`. Because the alphabet has 26 letters, applying it twice returns the original text, which makes the same operation both the encoding and the decoding step. Like Base64, it offers no protection: the shift is the entire key, and it is public.

### Using `tr`

`rot13` was not installed on the server, so I used `tr`, which replaces each character in a first set with the character in the same position of a second set. Ranges are expanded into lists before the mapping is applied.

| Attempt | Command | Result |
|---|---|---|
| 1 | `tr [A-Za-z] [N-Mn-m]` | Error: a range must go from lower to higher, and `N-M` goes backwards |
| 2 | `tr [A-Za-z] [M-Nm-n]` | Output made almost entirely of `]` |
| 3 | `tr [A-Za-z] [N-ZA-Nn-za-m]` | `Tgd ozrrvnqc` instead of `The password` |
| 4 | `tr [A-Za-z] [N-ZA-Mn-za-m]` | Correct |

The third attempt was the most instructive. Uppercase letters were decoded correctly, but every lowercase letter was off by one. The block `A-N` contains 14 letters, not 13, so the second set had 53 characters against 52 in the first. The extra `N` took the position that should have mapped `a` to `n`, and every lowercase letter after it shifted by one. I only found this by counting the letters one by one; my first two estimates were wrong.

The second attempt had a different cause. In `tr`, square brackets are not part of the range syntax; they are treated as literal characters. `[A-Za-z]` is therefore 54 characters, while `[M-Nm-n]` is only 6. When the second set is shorter, `tr` repeats its last character, here `]`, to fill the gap. The brackets did no harm in the final command only because `[` and `]` sat in matching positions in both sets. Unquoted brackets can also be expanded by the shell as a filename pattern, so the correct form uses quotes and no brackets:

```bash
tr 'A-Za-z' 'N-ZA-Mn-za-m' < data.txt
```

**Takeaway:** `tr` maps characters by position, not by intent. Any tool that works this way fails through misalignment, and the typical symptom is output that is almost right and drifts from a certain point onward. When that happens, check the counts instead of assuming them.

**Next:** Bandit level 12.
