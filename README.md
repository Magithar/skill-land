<h1 align="center">skill-land</h1>

<p align="center">
  <img src="https://img.shields.io/badge/license-MIT-blue.svg" alt="License"/>
  <a href="https://github.com/Magithar/skill-land/releases"><img src="https://img.shields.io/github/v/release/Magithar/skill-land?label=release" alt="Release"/></a>
  <a href="https://www.npmjs.com/package/skill-land"><img src="https://img.shields.io/npm/v/skill-land" alt="npm version"/></a>
  <a href="https://www.npmjs.com/package/skill-land"><img src="https://img.shields.io/npm/dt/skill-land" alt="npm downloads"/></a>
  <a href="https://github.com/Magithar/skill-land/actions/workflows/check.yml"><img src="https://github.com/Magithar/skill-land/actions/workflows/check.yml/badge.svg" alt="CI"/></a>
</p>

<p align="center">
  Installs an agent skill to the directory the agent <b>actually reads</b>, then reads every file back and exits non-zero if it didn't land. Built because <code>npx skills add -g</code> reports success while writing somewhere 13 agents never look. It also audits skills you already have, whatever installed them, and can ask a fresh agent session whether it really sees the skill.<br/><br/>
  Works with <a href="#supported-agents">Claude Code</a>, <a href="#supported-agents">OpenAI Codex</a>, <a href="#antigravity">Antigravity</a>, <a href="#supported-agents">Gemini CLI</a>, <a href="#supported-agents">Cursor</a>, <a href="#supported-agents">GitHub Copilot</a> and <a href="#supported-agents">more</a>.
</p>

---

<p align="center">
  <a href="#install">Install</a> • <a href="#usage">Usage</a> • <a href="#why-this-exists">Why</a> • <a href="#what-verified-means">Verified</a> • <a href="#auditing-a-whole-directory">Auditing</a> • <a href="#does-the-agent-actually-load-it---discover">Discover</a> • <a href="#security-scan">Security Scan</a> • <a href="#antigravity">Antigravity</a> • <a href="#skill-discovery">Skill Discovery</a> • <a href="#what-skill-land-is-not">Is Not</a> • <a href="#project-structure">Project Structure</a> • <a href="#releasing">Releasing</a> • <a href="#related">Related</a>
</p>

---

## Install

No install step. Run it with `npx`:

```bash
npx skill-land Magithar/SKILLmama --for codex,antigravity   # install, then verify
npx skill-land Magithar/SKILLmama --for codex --verify      # audit an existing install
```

The examples throughout use [SKILLmama](https://github.com/Magithar/SKILLmama), a capability
discovery skill by the same author. Any GitHub repo, local folder or `SKILL.md` works as a source.

Or install it globally:

```bash
npm install -g skill-land
```

Node 18 or later. No dependencies.

### Supported agents

| Agent | `--for` | Upstream `skills add -g` |
|-------|---------|--------------------------|
| Claude Code | `claude-code` | works |
| OpenAI Codex | `codex` | writes to the wrong directory |
| Antigravity | `antigravity` | writes to the wrong directory ([details](#antigravity)) |
| Antigravity CLI | `antigravity-cli` | writes to the wrong directory |
| Gemini CLI | `gemini-cli` | writes to the wrong directory |
| Cursor | `cursor` | writes to the wrong directory |
| GitHub Copilot | `github-copilot` | writes to the wrong directory |
| opencode | `opencode` | writes to the wrong directory |
| Firebender | `firebender` | writes to the wrong directory |
| Amp | `amp` | writes to the wrong directory |

`skill-land --list` prints every agent with the directories it writes to.

---

## Usage

```bash
npx skill-land <source> --for <agent> [options]
```

`<source>` is `owner/repo`, a GitHub URL, a raw `.md` URL, or a local path.

| Option | Effect |
|---|---|
| `--for <agent>` | target agent; repeatable or comma-separated |
| `--skill <name>` | which skill, when the source contains several |
| `--all` | every skill in the source, not just one |
| `--verbose` | per-skill detail when `--all` (default: one line each) |
| `--project` | install into `./` instead of the user-global directory |
| `--dry-run` | resolve and report, write nothing |
| `--verify` | check an existing install, write nothing |
| `--discover` | also ask a fresh agent session to list its skills (claude-code, codex, gemini-cli, antigravity) |
| `--skip-security` | skip the SkillSpector scan (acknowledged, not silent) |
| `--strict` | refuse to install anything SkillSpector flags |
| `--list` | show known agents and their directories |

```bash
# install and verify
skill-land Magithar/SKILLmama --for codex,antigravity

# audit an install someone else did
skill-land Magithar/SKILLmama --for codex --verify

# audit every skill you have, however it was installed
skill-land ~/.agents/skills --for antigravity --verify --all
```

---

## Why this exists

`npx skills add -a <agent> -g` reports success while writing to the wrong directory
for 13 agents, including Codex, Antigravity, Cursor, GitHub Copilot, Gemini CLI and
opencode. Reproduced against `skills@1.5.22` (latest) on 2026-08-08:

```
$ npx skills add Magithar/SKILLmama -a codex -g -y
  copy → Codex
  ✓ SKILLmama (copied)
    → ~/.agents/skills/skillmama
  Done!
$ echo $?
0
$ ls ~/.codex/skills/
                        # empty. Codex never sees it.
```

The correct path is in the CLI's own registry and never read. `isUniversalAgent()`
(`dist/cli.mjs:1678`) classifies any agent whose *project* directory is
`.agents/skills` as "universal", and `getAgentBaseDir()` then routes it to the
canonical store, discarding `globalSkillsDir`:

```js
function isUniversalAgent(type) {
	return agents[type].skillsDir === ".agents/skills";
}
```

Every failure of this kind exits 0. Nothing in the ecosystem checks the install took.

A second, separate bug: `-a claude-code,antigravity` is rejected as "Invalid agents"
while both names appear in the valid list it prints.

---

## What "verified" means

A skill is a **directory**, not a file. `skill-land` copies every file under it and reads
each one back. For each file:

1. file exists at the target path
2. non-empty, unless the source file is empty too
3. byte-identical to the source
4. frontmatter `name` matches the source (`SKILL.md` only)

Any failure prints the specific check that failed and **exits non-zero**. A write that
throws (permissions, read-only directory) is reported as a failed install, not a crash.

Self-contained skills are unaffected — the directory holds one file, so one file is copied.
Multi-file skills are the reason this matters: `impeccable` ships 148 files, and an install
of only its `SKILL.md` leaves every `node .../scripts/context.mjs` reference dangling.

Where an agent has several directories it reads, `--verify` counts the skill as installed if
it is **complete in any one of them**. Requiring every fallback to hold a copy reports
working installs as broken.

### Leftover files

Installing copies over the top and never deletes, so a file dropped from a skill stays on disk
after a reinstall. `--verify` lists anything in the installed directory that the source does not
have:

```
OK    ~/.codex/skills/pack  3/3 files  (verify only)
WARN  ~/.codex/skills/pack  1 leftover file not in the source: scripts/old.sh
```

This is a warning and does not change the exit code, because an installed directory can also hold
files a person or the agent added, and nothing here can tell those from stale ones. `skill-land`
never deletes them. It is skipped for a raw `.md` source, which only knows about `SKILL.md`.

### What it cannot tell you

Verification assumes the installer **copied**. Some installers **rewrite** on install — the
`skills` CLI adapts `impeccable` for Antigravity, changing `.agents/skills/...` paths to
`.gemini/skills/...` and `$impeccable` to `/impeccable`. Those files are correct and will
still report `content matches source` failures, because they genuinely differ from the source.
`skill-land` reports the difference and does not guess which side is right.

**Renamed and removed skills are not detected.** `skill-land` keeps no record of what it installed,
and the install directory comes from the skill's `name`. After a rename the old directory stays
next to the new one, and `--verify` only looks at the new one. A skill removed from its source is
never looked at. There is no uninstall command, so delete the old directory by hand.

Tested against: truncated file, wrong skill in the right place, missing file,
unwritable destination, and a real `npx skills add` install (correctly reported FAIL).

---

## Auditing a whole directory

`--all` takes every skill in the source instead of one. It is never the default:
[`selectSkill()`](skill-land.mjs) refuses to guess when a source holds several,
and `--all` is you saying you meant all of them.

The use it was built for is `--verify --all` — checking whether the skills you
already installed are where the agent actually reads, whatever put them there.
That question is unanswerable today even if your setup works, because nothing in
the ecosystem reads an install back:

```
$ skill-land ~/.agents/skills --for antigravity --verify --all

  source   /Users/you/.agents/skills
  skills   1 found
  impeccable                    FAIL  0/3
        x ~/.gemini/config/skills/impeccable/SKILL.md
          file exists: ~/.gemini/config/skills/impeccable/SKILL.md
        x ~/.gemini/skills/impeccable/SKILL.md
          content matches source: 10401 bytes sha abd76e36cf1d vs 10401 bytes sha a1ea82ce80f4
  1 skill, 3 failed
```

That middle line is why the check compares hashes and not sizes: both files are
10401 bytes and they are not the same file. A copy that drifted after install
looks identical to `ls`.

One line per skill, because a report nobody reads verifies nothing; `--verbose`
restores the full per-target output. Failing paths are always named — a summary
you have to re-run to act on is not a summary.

With `--strict`, a flagged skill is skipped and the rest still install; the run
exits non-zero and lists what it skipped. All-or-nothing across a directory would
let one flagged skill block every clean one.

---

## Does the agent actually load it? `--discover`

Everything above proves the bytes are on disk. It does not prove the agent loads them: it may
need a restart, reject the frontmatter, or read the directory only in some workspaces.
`--discover` starts a fresh non-interactive session from a neutral directory and checks that the
skill name appears in what the agent says it can see.

```bash
skill-land Magithar/SKILLmama --for codex,antigravity --discover
```

| Agent | Listing used | Kind |
|---|---|---|
| Antigravity | `agy -p /skills` | a real skill list |
| Gemini CLI | `gemini skills list` | a real skill list, with enabled/disabled state |
| Codex | `codex exec "list your skills"` | asks the model |
| Claude Code | `claude -p "list your skills"` | asks the model |

Each was tested by planting a uniquely named canary skill and confirming it appeared, then with
real skills from git. A skill that is listed but switched off (Gemini CLI prints `[Disabled]`)
fails: on disk and listed is not the same as loaded. Every other agent reports `NOT CHECKED`,
never a pass: an unknown is not a success. A missing binary or a timeout is also `NOT CHECKED`;
only a completed listing that lacks the skill fails the run.

The Codex and Claude Code listings go through the model, so they cost an API call and a `FAIL`
is a lead to investigate, not proof. It is opt-in for that reason, and skipped with `--project`.

---

## Security scan

Skills run with full agent permissions, and this tool writes a file fetched from a URL into a
directory an agent auto-loads. So it scans before writing, delegating to
[NVIDIA/SkillSpector](https://github.com/NVIDIA/SkillSpector) rather than reimplementing it:

```bash
uv tool install git+https://github.com/NVIDIA/skillspector.git
```

### What always runs: disclosure

A skill is instructions an agent executes with full permissions, so before writing anything
`skill-land` shows what is actually in the file. No dependencies, so it runs for everyone:

```
contains
    3  shell commands        curl, rm, git
    1  network hosts         attacker.example.com
    5  credential refs       AWS_SECRET_ACCESS_KEY, ~/.ssh, id_rsa, ~/.aws
    2  destructive commands  rm -rf, git push --force
```

A skill with nothing notable says so:

```
contains  no shell commands, network calls or credential references
```

**This is disclosure, not a verdict.** It counts and names; it never scores and never prints
"safe". That distinction is deliberate. A built-in matcher that declared a skill safe would be
worse than no check, and the numbers below show why. Counting stays honest at any false-positive
rate because it makes no claim.

### What runs when available: SkillSpector

**It reports, it does not block.** That is a measured decision, not a shortcut.

Scanning 18 known-good skills already installed on a dev machine (2026-08-08, SkillSpector
v2.8.1, `--no-llm`):

| Verdict | Count |
|---|---|
| SAFE | 1 |
| CAUTION | 9 |
| **DO_NOT_INSTALL** | **8 (44%)** |

The failure mode is worth understanding, because it is not random. A skill that *documents* attack
patterns gets flagged for *containing* them: the static matcher has no negation handling, so a rule
saying "reject skills that bypass safety checks" reads identically to a skill saying "bypass safety
checks".

[SKILLmama](https://github.com/Magithar/SKILLmama) was a worked example. Its security checklist
scored `CRITICAL` until four DISCARD criteria were reworded to say the same thing in different
words; it now scores `7 / LOW / SAFE`. Its behaviour never changed, only its vocabulary. Adding an
explicit user-consent section was also tested and made the score **worse**, because describing a
safeguard trips the same patterns as describing the danger.

A gate that blocks 44% of legitimate skills does not make anyone safer; it teaches people to pass
the override flag reflexively. So findings are surfaced with file and line, and the decision stays
with the user. `--strict` blocks on a flag for anyone who wants that.

| SkillSpector verdict | Default | `--strict` |
|---|---|---|
| `SAFE` | installs | installs |
| `CAUTION` | installs, findings shown | installs |
| `DO_NOT_INSTALL` | installs, findings shown | **refuses, exit 1** |
| scanner error | `treated as NOT scanned` | same |
| not installed | `NOT SCANNED` + install hint | same |

### Baselines are reported, never applied

A skill repo can ship `.skillspector-baseline.yaml` to suppress findings. This installer scans
**without** it, deliberately: a baseline that travels with untrusted code would let a malicious
skill silence its own detections. SkillSpector does not auto-discover baselines either (verified:
a baseline in the scanned directory has no effect unless `--baseline` is passed), so honouring one
found in a fetched repo would be strictly less safe than the default.

Instead its presence is surfaced as a fact about the skill, which is itself a signal:

```
security SAFE  SAFE, LOW, score 7, 2 issue(s)
         NOTE this repo ships a baseline suppressing 2 finding(s).
              Scanned WITHOUT it: a baseline from an untrusted repo can hide real findings.
```

`--no-llm` is always passed. Without it SkillSpector requires an LLM provider and exits 2 with no
credentials, which would silently degrade this gate to a no-op for anyone without an API key. A
check that did not run must never read as a check that passed.

---

## Antigravity

Antigravity loads skills at startup, so an install cannot be confirmed live. The
`agy` search path was stated by an Antigravity maintainer on 2026-09-05 in
[antigravity-cli#103](https://github.com/google-antigravity/antigravity-cli/issues/103#issuecomment-5547952239):

```
~/.gemini/config/skills            primary; independently confirmed by live testing
~/.gemini/skills                   fallback
~/.gemini/antigravity-cli/skills   fallback
```

`--for antigravity` writes to all three and says so. The `language_server` binary
contains `skillsPaths` and `skills_paths`, so this is a configurable list rather than a
single directory, and the maintainer statement covers the CLI — whether the IDE surface
reads the same list is **unverified**.

Checked against the shipped binaries (Antigravity 2026-07-30 build, inspected
2026-09-05). `language_server` contains **no** `.gemini/*/skills` literal at all:
`skills_paths` is a repeated protobuf field, so the client passes the list in, and
skills resolve to `skills/<name>/` under a *customization root*. Both roots we write
to are named in the shipped strings and config:

- `~/.gemini/config/` holds `config.json` and `skills/` — the global root
- `~/.gemini/antigravity-cli/` — "The CLI is configured via
  `~/.gemini/antigravity-cli/settings.json`" (`language_server`)

1.1.0 and earlier also wrote `~/.gemini/antigravity/skills`, on the reasoning that
Antigravity created that directory itself. That reasoning was wrong: `app.asar` defines
`~/.gemini/antigravity` as `IDE_OLD_DATA_DIR`, the **legacy IDE data directory** being
migrated to `~/.gemini/antigravity-ide`. It was never a skills root. No longer written.

### Live verification

Confirmed against **`agy` 1.1.27 on 2026-09-05**, not inferred. A uniquely-named
canary skill was planted in each candidate directory and `agy -p "/skills"` run from
a neutral workspace:

| directory | listed? |
|---|---|
| `~/.gemini/config/skills` | **yes** |
| `~/.gemini/skills` | **yes** |
| `~/.gemini/antigravity-cli/skills` | **yes** |
| `~/.agents/skills` | no |
| `~/.gemini/antigravity/skills` | no |

All three directories `skill-land` writes to are read. The two it does not write to
are not.

`~/.agents/skills` appears **only** when its parent is the active workspace
(`agy --add-dir $HOME`) — that is project-local behaviour, not a global search path,
which is exactly what [#103](https://github.com/google-antigravity/antigravity-cli/issues/103)
is asking to change. Confirmed still open at 1.1.27.

Two side findings: project-local `.agents/skills` is read only when the directory is
a **registered workspace** — being the current directory was not enough — and
`.antigravity/skills` and `.agy/skills` are not read at all.

Restart Antigravity fully after installing.

---

## Skill discovery

Walks up to 10 levels, skipping only `.git`, `node_modules`, `.venv` and
`__pycache__`, with a symlink-loop guard. Project-specific folder names are
deliberately not hardcoded: a repo whose skill lives in `assets/` is legal, and
skipping it produced a false "no SKILL.md found".

When a source contains more than one skill the tool **refuses to guess**. It lists
what it found and exits non-zero until you pass `--skill <name>`:

```
$ skill-land product-on-purpose/pm-skills --for codex
  error  78 skills found; refusing to guess.
         Pick one with --skill <name>:
           define-hypothesis    ./skills/define-hypothesis/SKILL.md
           ...
```

---

## What skill-land Is Not

- Not a registry or a package manager
- Not a publisher
- Not a content transformer: no per-platform rewrites

skill-land **installs and verifies**. One `SKILL.md` works across all of these agents; the failure
mode here is a wrong-path bug, not a content-format problem.

---

## Project Structure

```
skill-land.mjs              the CLI: resolve, scan, copy, read back, discover
agents.json                 every agent's skill directories; broken: true marks upstream misroutes
test.mjs                    test suite (node test.mjs)
CHANGELOG.md                release notes, one section per version
.github/workflows/
  check.yml                 runs the tests on every push and pull request
  publish.yml               publishes to npm on a v* tag
```

### Adding an agent

Edit `agents.json`. `broken: true` marks agents where the upstream CLI misroutes a
global install, which is what drives the warning note.

---

## Releasing

Published from CI by [trusted publishing](https://docs.npmjs.com/trusted-publishers), so there is
no `NPM_TOKEN` in this repo and the npm account keeps 2FA on write actions. npm mints a
short-lived token from the GitHub identity of `.github/workflows/publish.yml`.

```bash
# bump version in package.json, add a CHANGELOG section, commit, then:
git tag v1.8.0
git push origin v1.8.0
```

CI runs the tests, refuses to publish a version that already exists, checks the tag matches
`package.json`, publishes, and creates the GitHub release from that version's CHANGELOG section.

Releases up to 1.7.0 were made from [Magithar/skills](https://github.com/Magithar/skills), where
skill-land used to live under `tools/skill-land/`.

---

## Related

- [SKILLmama](https://github.com/Magithar/SKILLmama), capability discovery engine: finds and ranks
  the libraries, SDKs and tools for your stack. The skill used in the examples above.
- [Magithar/skills](https://github.com/Magithar/skills), agent skills installable as a Claude Code
  plugin, including `dep-egress` and `evidence-loop`.

---

Made by [@magithar](https://x.com/magithar). Follow along for new skills and tools.
